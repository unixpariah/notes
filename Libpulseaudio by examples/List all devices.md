1. Create new handler
- Allocate property list and append new string entry to the property list

```rust
let mut proplist = Proplist::new().ok_or("")?;
proplist
	.set_str(
		pulse::proplist::properties::APPLICATION_NAME,
		"SinkController",
		)
		.unwrap();
```

- Allocate mainloop, instantiate a new connection context and connect the context to the specified server

```rust
let mut mainloop = Mainloop::new().ok_or("")?;
let mut context = Context::new_with_proplist(&mainloop, "MainConn", &proplist).unwrap();
context.connect(None, pulse::context::FlagSet::NOFLAGS, None).unwrap();
```

- Iterate until the context status connection is Ready

```rust
loop {
	match mainloop.iterate(true) {
		IterateResult::Err(e) => {
			return Err(e.into());
		}
		IterateResult::Success(_) => {}
		IterateResult::Quit(_) => {
			return Err("Iterate state quit without an error".into());
		}
	}

	match context.get_state() {
		pulse::context::State::Ready => break,
		pulse::context::State::Failed | pulse::context::State::Terminated => {
			return Err("Context state failed/terminated without an error".into());
		}
		_ => {}
	}
}
```

-  Get the introspect from context, making the Handler ready


```rust
let introspect = context.introspect();

let handler = Handler {
	introspect,
	mainloop,
	context
}
```

2. List all available devices
- Create helper function wait_for_operation, it should take Operation and handler as its arguments, it will iterate until operation.get_state returns State::Done enum

```rust
fn wait_for_operation<G: ?Sized>(
	handler: Handler,
	op: Operation<G>,
    ) -> Result<(), Box<dyn crate::Error>> {
		loop {
			match handler.mainloop.iterate(true) {
				IterateResult::Err(e) => return Err(e.into()),
                IterateResult::Success(_) => {}
                IterateResult::Quit(_) => {}
            }
            match op.get_state() {
                State::Done => {
                    break;
                }
                State::Running => {}
                State::Cancelled => {
                    return Err("Operation cancelled without an error".into());
                }
            }
        }
    Ok(())
}
```

- Finally list all the devices

```rust
let list = Rc::new(RefCell::new(Vec::new()));
{
	let list = list.clone();
    let op = handler.introspect.get_sink_info_list(
	            move |sink_list: ListResult<&introspect::SinkInfo>| {
                if let ListResult::Item(item) = sink_list {
                    list.borrow_mut().push((item.index, item.volume));
                }
            });
    wait_for_operation(handler, op)?;
}
let device_list = list.borrow_mut().to_vec() // Vec<(u32, ChannelVolumes)
device_list.iter().for_each(|device| println!("{}", device.1.print()));
```