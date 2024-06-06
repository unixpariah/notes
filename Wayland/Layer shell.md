Writing simple layer shell client in zig

First we begin by installing the necessary dependencies

- **Zig-wayland**: Bindings and scanner for wayland.

```bash
zig fetch --save https://codeberg.org/ifreund/zig-wayland/archive/v0.1.0.tar.gz
```

- **Wayland Compositor**: This should implement the Layer Shell protocol. Compositors based on `wlroots` typically meet this requirement.
- **Wayland**: The protocol itself.
- **Wayland-Protocols**: This package contains additional protocols that extend the functionality of the core Wayland protocols. It's necessary for accessing features not available in Wayland's core.
- **Wayland-Scanner**: This is a tool used to generate C code from Wayland protocol specification files, which are written in XML. It's essential for creating the server and client APIs from the protocol specifications.

**NOTE**: The names of these dependencies are as they appear in the Nix package manager. The exact names may vary depending on your Linux distribution and its package manager.

Next, we need to get the XML file for the Layer Shell as it's not included in the core protocols. The Layer Shell XML file is used to generate the server and client APIs.

You can obtain the XML file for the Layer Shell from the following link: [wlr-layer-shell-unstable-v1](https://github.com/swaywm/wlr-protocols/blob/master/unstable/wlr-layer-shell-unstable-v1.xml). Place the xml file in ./protocols/wlr-layer-shell-unstable-v1.xml

Let's start with a simple `build.zig`:

```c
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "foo",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);
}
```

Now, let's modify it to handle Wayland:

```c
const std = @import("std");

// Import the Wayland scanner from zig-wayland
const Scanner = @import("zig-wayland").Scanner;

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // Create a new Wayland scanner
    const scanner = Scanner.create(b, .{});
    // Create a new module for the generated Wayland code
    const wayland = b.createModule(.{ .source_file = scanner.result });

    // Add the necessary protocols.
    // We'll add the Layer Shell protocol from the project root directory,
    // and the XDG Shell protocol from the system (requirement for xdg-popup)
    scanner.addCustomProtocol("protocols/wlr-layer-shell-unstable-v1.xml");
    scanner.addSystemProtocol("stable/xdg-shell/xdg-shell.xml");

    // Generate headers for the necessary protocols.
    // Pass the maximum version implemented by your Wayland server or client.
    // Requests, events, enums, etc. from newer versions will not be generated,
    // ensuring forwards compatibility with newer protocol XML.

    // Needed to request a surface from the compositor
    scanner.generate("wl_compositor", 1);
    // For this simple example, we'll use shared memory to draw to the surface
    //  instead of the GPU
    scanner.generate("wl_shm", 1);
    // Generate the Layer Shell headers
    scanner.generate("zwlr_layer_shell_v1", 4);
	// Generate the Output headers
    scanner.generate("wl_output", 4);

    const exe = b.addExecutable(.{
        .name = "foo",
        .root_source_file = .{ .path = "src/main.zig" },
        .target = target,
        .optimize = optimize,
    });

    // Add the generated Wayland code as a module to the executable
    exe.addModule("wayland", wayland);
    // Link against the C standard library and the Wayland client library
    exe.linkLibC();
    exe.linkSystemLibrary("wayland-client");

    // Remove when https://github.com/ziglang/zig/issues/131 is implemented
    scanner.addCSource(exe);

    b.installArtifact(exe);
}
```

This `build.zig` script sets up the build environment for a Wayland client application. It uses the Wayland scanner to generate C code from the Wayland protocol XML files, and then adds this generated code to the build.

Your project directory should now look somewhat like this:

```bash
.
├── build.zig
├── build.zig.zon
├── protocols
│   └── wlr-layer-shell-unstable-v1.xml
└── src
    └── main.zig
```

Finally we can proceed to writing the actual code.

Let's start with connecting to wayland and getting the registry which we will use to bind globals.

```c
// Import wayland dependency
const wayland = @import("wayland");
// This contains all wayland core protocols, we alias it for convenience
const wl = wayland.client.wl;

pub fn main() !void {
    const display = try wl.Display.connect(null);
    const registry = try display.getRegistry();
}
```

Let's create a struct containing

```c
// This contains all protocols marked as unstable (z) from wlroots (wlr)
const zwlr = wayland.client.zwlr;

const State = struct {
    shm: ?*wl.Shm,
    compositor: ?*wl.Compositor,
    layer_shell: ?*zwlr.LayerShellV1,

    const Self = @This();

    fn new() Self {
        return .{ // Initialize as null and populate later
            .shm = null,
            .compositor = null,
            .layer_shell = null,
        };
    }
};
```

Now let's create a callback method for registry events which will be needed to bind the globals (shm, compositor, layer_shell). The registry events are emitted by the server for each available global, which then can be bound.

```c
fn registryListener(registry: *wl.Registry, event: wl.Registry.Event, state: *State) void {
    switch (event) {
        .global => |global| {
            if (mem.orderZ(
                u8,
                global.interface,
                wl.Compositor.getInterface().name) == .eq
            ) {
                state.compositor = registry.bind(
                    global.name,
                    wl.Compositor,
                    wl.Compositor.generated_version
                ) catch return;
            } else if (mem.orderZ(
                u8,
                global.interface,
                wl.Shm.getInterface().name) == .eq
            ) {
                state.shm = registry.bind(
                    global.name,
                    wl.Shm,
                    wl.Shm.generated_version
                ) catch return;
            } else if (mem.orderZ(
                u8,
                global.interface,
                zwlr.LayerShellV1.getInterface().name) == .eq
            ) {
                state.layer_shell = registry.bind(
                    global.name,
                    xdg.WmBas,
                    zwlr.LayerShellV1.generated_version
                ) catch return;
            }
            else if (mem.orderZ(
                u8,
                global.interface,
                wl.Output.getInterface().name) == .eq
            ) {
	            // Not needed at the moment
            }
        },
        .global_remove => {},
    }
}
```

To make it slightly more readable lets create an enum of globals, which we can then use to turn the global interface into enum and switch on it.

```c
const EventInterfaces = enum {
    wl_shm,
    wl_compositor,
    zwlr_layer_shell_v1,
    wl_output,
};

fn registryListener(registry: *wl.Registry, event: wl.Registry.Event, state: *State) void {
    switch (event) {
        .global => |global| {
            const events = std.meta.stringToEnum(
                EventInterfaces,
                std.mem.span(global.interface)
            ) orelse return;
            switch (events) {
                .wl_compositor => {
                    state.compositor = registry.bind(
                        global.name,
                        wl.Compositor,
                        wl.Compositor.generated_version
                    ) catch return;
                },
                .wl_shm => {
                    state.shm = registry.bind(
                        global.name,
                        wl.Shm,
	                    wl.Shm.generated_version
                    ) catch return;
                },
                .zwlr_layer_shell_v1 => {
                    state.layer_shell = registry.bind(
                        global.name,
                        zwlr.LayerShellV1,
                        zwlr.LayerShellV1.generated_version
                    ) catch return;
                },
                .wl_output => {
                    // Not needed atm
                },
            }
        },
        .global_remove => {},
    }
}
```

Now let's start the registry listener and the main loop.

```c
pub fn main() !void {
    // ... existing code

    var state = State.new();
	// Pass in the state pointer to mutate the contents of struct
    registry.setListener(*State, registryListener, &state);
    // Ensure that all events sent to the compositor were processed
    if (display.roundtrip() != .SUCCESS) return error.DispatchFailed;
    // Start the event loop, dispatch events from compositor on each iteration
    while (display.dispatch() == .SUCCESS) {}
}
```

Create a callback function for layer surface configure event

```c
pub fn layerSurfaceListener(lsurf: *zwlr.LayerSurfaceV1, event: zwlr.LayerSurfaceV1.Event, state: *State) void {
    switch (event) {
        .configure => |configure| {
            state.layer_surface.setSize(configure.width, configure.height);
            state.layer_surface.ackConfigure(configure.serial);
        },
        .closed => {},
    }
}
```

What's left now is binding the wl_output and handling the surfaces.

```c
// ...
.wl_output => {
    // Bind the wl_output global
    const global_output = registry.bind(
        global.name,
        wl.Output,
        wl.Output.generated_version,
    ) catch return;

    // Ask compositor to create a new surface
    const surface = state.compositor.?.createSurface() catch return;

    // Create layer surface for an existing surface. Assign it to overlay layer, 
    // which is the top-most layer
    const layer_surface = state.layer_shell.?.getLayerSurface(surface, global_output, .overlay, "foo") catch return;
    layer_surface.setListener(*State, layerSurfaceListener, state);
    // As it's gonna be a full screen surface, anchor it to all edges and 
	// corners.
    layer_surface.setAnchor(.{
        .top = true,
        .right = true,
        .bottom = true,
        .left = true,
    });
    layer_surface.setExclusiveZone(-1);
    // Layer shell requires that you commit to it once before actually attaching 
    // buffer to it.
    surface.commit();
},
// ...
```

After binding the output everything for drawing is ready. Let's create a draw method on State.

```c
fn draw(self: *Self, width: u32, height: u32) !void {
    const stride = width * 4;
    const total_size = stride * height;

    // Create file descriptor
    const fd = try posix.memfd_create("foo", 0);
    defer std.posix.close(fd);
    // Resize file descriptor
    posix.ftruncate(fd, @intCast(stride)) catch @panic("OOM");

    // Create pool with file descriptor as a backing memory for the pool.
    const pool = try self.shm.?.createPool(fd, @intCast(total_size));
    defer pool.destroy();

    // Map file descriptor into memory
    const mmap = try posix.mmap(null, total_size, posix.PROT.READ | 
	    posix.PROT.WRITE, posix.MAP{ .TYPE = .SHARED }, fd, 0);

    // Fill mmap with 255 to display a white surface
    @memset(mmap, 255);

    // Create buffer for the content of wl_surface
    const buffer = try pool.createBuffer(0, @intCast(width), @intCast(height), 
		@intCast(stride), wl.Shm.Format.argb8888);

    // Damage surface to tell compositor which part of surface needs to be 
    // redrawn.
    self.surface.damage(0, 0, width, height)

    // Set buffer as content of the surface
    self.surface.attach(self.buffer, 0, 0);

    // Finally commit pending surface state
    self.surface.commit();
}
```

As this is a simple example, we'll call draw inside of configure event

```c
pub fn layerSurfaceListener(lsurf: *zwlr.LayerSurfaceV1, event: zwlr.LayerSurfaceV1.Event, state: *State) void {
    switch (event) {
        .configure => |configure| {
            state.layer_surface.setSize(configure.width, configure.height);
            state.layer_surface.ackConfigure(configure.serial);
            state.draw(configure.width, configure.height);
        },
        .closed => {},
    }
}
```

And that's how you can draw a white fullscreen layer
