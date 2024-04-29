Writing simple layer shell client in zig

First we begin by installing the necessary dependencies

- **Zig-wayland**: Bindings and scanner for wayland.

```bash
git clone https://github.com/ifreund/zig-wayland.git ./deps/zig-wayland
# In project root directory
```

- **Wayland Compositor**: This should implement the Layer Shell protocol. Compositors based on `wlroots` typically meet this requirement.
- **Wayland**: The protocol itself.
- **Wayland-Protocols**: This package contains additional protocols that extend the functionality of the core Wayland protocols. It's necessary for accessing features not available in Wayland's core.
- **Wayland-Scanner**: This is a tool used to generate C code from Wayland protocol specification files, which are written in XML. It's essential for creating the server and client APIs from the protocol specifications.

**NOTE**: The names of these dependencies are as they appear in the Nix package manager. The exact names may vary depending on your Linux distribution and its package manager.

Next, we need to download the XML file for the Layer Shell as it's not included in the core protocols. The Layer Shell XML file is used to generate the server and client APIs.

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
const Scanner = @import("deps/zig-wayland/build.zig").Scanner;

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
├── protocols
│   └── wlr-layer-shell-unstable-v1.xml
├── src
│   └── main.zig
└── deps
    └── zig-wayland
        ├── build.zig
        ├── example
        │   ├── globals.zig
        │   ├── listener.zig
        │   ├── list.zig
        │   └── seats.zig
        ├── LICENSE
        ├── README.md
        └── src
            ├── client_display_functions.zig
            ├── common_core.zig
            ├── ref_all.zig
            ├── scanner.zig
            ├── test_data
            │   └── wayland.xml
            ├── wayland_client_core.zig
            ├── wayland_server_core.zig
            └── xml.zig
```

Finally we can proceed to writing the actual code.

Let's start with connecting to wayland and getting the registry (Registry is used to display all protocols implemented by your compositor)

```c
const wayland = @import("wayland"); // Import wayland dependency
// This contains all wayland core protocols, we alias it for convenience
const wl = wayland.client.wl;

pub fn main() !void {
    const display = try wl.Display.connect(null);
    const registry = try display.getRegistry();
}
```

As Wayland is object oriented `Wayland is object-oriented: All communication between a client and a server is formulated in the form of method calls on some objects` we'll have to create a struct holding all of our necessary wayland protocols

```c
// This contains all protocols marked as unstable (z) from wlroots (wlr)
const zwlr = wayland.client.zwlr;

const Context = struct {
    shm: ?*wl.Shm,
    compositor: ?*wl.Compositor,
    layer_shell: ?*zwlr.LayerShellV1,
    fn new() Context {
        return Context { // Initialize as null and populate later
            .shm = null,
            .compositor = null,
            .layer_shell = null,
        };
    }
};
```

Now that we have the struct, let's create a callback method for registry events which will be needed to bind the globals (shm, compositor, layer_shell). The registry events are emitted by the server for each available global, which then can be bound. [Binding to globals](https://wayland-book.com/registry/binding.html)

```c
fn registryListener(registry: *wl.Registry, event: wl.Registry.Event, context: *Context) void {
    switch (event) {
        .global => |global| {
            if (mem.orderZ(
                u8,
                global.interface,
                wl.Compositor.getInterface().name) == .eq
            ) {
                context.compositor = registry.bind(
                    global.name,
                    wl.Compositor,
                    wl.Compositor.generated_version
                ) catch return;
            } else if (mem.orderZ(
                u8,
                global.interface,
                wl.Shm.getInterface().name) == .eq
            ) {
                context.shm = registry.bind(
                    global.name,
                    wl.Shm,
                    wl.Shm.generated_version
                ) catch return;
            } else if (mem.orderZ(
                u8,
                global.interface,
                zwlr.LayerShellV1.getInterface().name) == .eq
            ) {
                context.layer_shell = registry.bind(
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

Yeah, it looks horrible, so to make it more readable let's create enum consisting of all of our necessary global.

```c
const EventInterfaces = enum {
    wl_shm,
    wl_compositor,
    zwlr_layer_shell_v1,
    wl_output,
};
```

And now we can simplify the callback method

```c
fn registryListener(registry: *wl.Registry, event: wl.Registry.Event, context: *Cpntext) void {
    switch (event) {
        .global => |global| {
            const events = std.meta.stringToEnum(
                EventInterfaces,
                std.mem.span(global.interface)
            ) orelse return;
            switch (events) {
                .wl_compositor => {
                    context.compositor = registry.bind(
                        global.name,
                        wl.Compositor,
                        wl.Compositor.generated_version
                    ) catch return;
                },
                .wl_shm => {
                    context.shm = registry.bind(
                        global.name,
                        wl.Shm,
	                    wl.Shm.generated_version
                    ) catch return;
                },
                .zwlr_layer_shell_v1 => {
                    context.layer_shell = registry.bind(
                        global.name,
                        zwlr.LayerShellV1,
                        zwlr.LayerShellV1.generated_version
                    ) catch return;
                },
                .wl_output => {
	                // Not needed at the moment
                },
            }
        },
        .global_remove => {},
    }
}
```

Now let's modify our main to actually create the struct and start the registry listener

```c
pub fn main() !void {
    // ... existing code

    var context = Context.new();
	// Pass in the context pointer to mutate the contents of struct
    registry.setListener(*Context, registryListener, &context);
}
```