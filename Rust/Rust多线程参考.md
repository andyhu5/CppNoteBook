```rust
// 一个完整的多线程参考：
use std::{
    fs::File,
    io::Write,
    sync::{Arc, Mutex},
    thread::{self, sleep},
};

use chrono::Local;

fn main() {
    let log = File::create("foo.txt").expect("failed to create file");

    let my_log = Arc::new(Mutex::new(log));

    let mut handles = Vec::new();

    for _i in 0..5 {
        let my_log = Arc::clone(&my_log);

        let handle = std::thread::spawn(move || {
            loop {
                {
                    let mut log = my_log.lock().unwrap();
                    let now = Local::now();

                    writeln!(
                        log,
                        "hello world :{:?} {}",
                        thread::current().id(),
                        now.format("%Y-%m-%d %H:%M:%S")
                    )
                    .unwrap();
                }
                sleep(std::time::Duration::from_secs(5));
            }
        });
        handles.push(handle);
    }
    for handle in handles {
        handle.join().unwrap();
    }
}

```

