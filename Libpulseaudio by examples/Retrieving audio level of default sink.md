1. Create new handler
- Allocate property list and append new string entry to the property list

```rust
 let mut proplist = Proplist::new().ok_or("")?;
proplist
	.set_str(
		pulse::proplist::properties::APPLICATION_NAME,
		"SinkController")
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

2. Get the default sink
- Create helper function wait_for_operation, it should take Operation as its only argument, iterate until operation.get_state returns State::Done enum

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

- Retrieve default sink name from the server info on introspect

```rust
let server: Rc<RefCell<Option<Option<String>>>> = Rc::new(RefCell::new(Some(None)));
{
	let server = server.clone();
    let op = handler.introspect.get_server_info(move |result| {
	    server
            .borrow_mut()         
            .replace(
	            result.default_sink_name.as_ref().map(|cow|cow.to_string()));
            });
            wait_for_operation(handler, op)?;
}

let default_sink_name = server.borrow_mut().take().flatten().unwrap();
```

- Retrieve default device volume with the default sink name

```rust
 let device = Rc::new(RefCell::new(None));
{
	let device = device.clone();
    let op = handler.introspect.get_sink_info_by_name(
			&default_sink_name,
            move |sink_list: ListResult<&introspect::SinkInfo>| {
	            if let ListResult::Item(item) = sink_list {
	                device.borrow_mut().replace(item.volume);
                }
            });
    wait_for_operation(handler, op)?;
}
let default_device_volume = device.borrow_mut().take().unwrap();
```

- Finally print the volume to get the human readable values

```rust
default_device_volume.print() // Returns volume level in % for each channel
```

3. Notes
- Implement graceful shutdown to avoid Address boundary error

```rust
impl Drop for Handler {
    fn drop(&mut self) {
        self.context.disconnect();
        self.mainloop.quit(pulse::def::Retval(0));
    }
}
```

4. Honorable mentions
- List all available devices

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
let device_list = list.borrow_mut().to_vec()
```

