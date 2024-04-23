1. Initiate threaded Mainloop and instantiate new connection context

```rust
 let mut mainloop = Mainloop::new().unwrap();
let mut context = Context::new(&mainloop, "volume-change-listener").unwrap();
context.connect(None, ContextFlagSet::NOFLAGS, None).unwrap();
```

2. Check current context state until the connection is ready

```rust
loop {
    if context.get_state() == libpulse_binding::context::State::Ready {
    break;
    }
}
```

3. Set subscribe callback of the connection context and subscribe with proper facility mask (in this case sink)

```rust
context.set_subscribe_callback(Some(Box::new(move |_, _, _| {
    println!("Volume level changed!");
})));
context.subscribe(InterestMaskSet::SINK, |_| {});
```

4. Mainloop and Context can't be dropped or the listener will stop so either keep them around or leak them

```rust
let mainloop = Box::new(mainloop);
let context = Box::new(context);
Box::leak(context);
Box::leak(mainloop);
```