# "Rinf copies flutter_rust_bridge, says bridge bad, claims rinf ultimate"'s appendix

Link: https://www.reddit.com/r/rust/comments/191b2to/rinf_copies_a_lot_from_flutter_rust_bridge_but/

Remark: The `cmp_frb` and `cmp_rinf` branches in this repository are used for git diff below.

## 0. Links in the main text

* [1] https://web.archive.org/web/20240104123133/https://rinf.cunarist.com/
* [2] https://web.archive.org/web/20240106005703/http://web.archive.org/screenshot/https://github.com/cunarist/rinf
* [3] https://web.archive.org/web/20231220063420/https://github.com/cunarist/rinf/discussions/104
* [4] https://web.archive.org/web/20240103061504/https://github.com/cunarist/rinf/discussions/195
* [5] https://www.reddit.com/r/rust/comments/18egmb4/comment/kcni0a2/?utm_source=share&utm_medium=web2x&context=3
* [6] https://www.reddit.com/r/FlutterDev/comments/18e8apx/comment/kclx8xy/?utm_source=share&utm_medium=web2x&context=3

## 1. Analysis of git diff

The git diff can be viewed here: https://github.com/fzyzcjy/ask_hackernews_appendix/compare/cmp_frb...cmp_rinf?expand=1 (scroll down a bit). The left-hand side is flutter_rust_bridge, and the right-hand side is rinf.

Here is an analysis on the difference:
* Delete comments which mentioned flutter_rust_bridge. (load.io.dart, into_into_dart.rs)
* Change a few lines of the `spawn_bridge_task` to make it non-spawn, i.e. runs directly on current thread. The same effect can be done without having to copy and modify the bridge.
* Rename things, change imports, exports, remove annotations and functions, change comments, change text, etc.
* Remark: The `worker.rs`, though seemingly new in git diff, is still copied from pool.rs, and only removed comments, changed formats, and remove error handling ([see this](https://www.diffchecker.com/FYTzoY8n/) for detailed diff)

P.S. Detailed steps to prepare the diff page:
* Switch to `cmp_frb` branch ("frb" means "flutter_rust_bridge"), copy-paste from flutter_rust_bridge v1.79.0, and commit git.
    * Folders: frb_dart/lib/src, frb_rust/src
* Switch to `cmp_rinf` branch, copy-paste from rinf master branch, and commit git.
    * Folders: flutter_ffi_plugin/lib/src/engine, rust_crate/src/engine
* Create a GitHub pull request between `cmp_frb` and `cmp_rinf` branches

## 2. Rinf can be much more convenient for users

The example code in rinf's website frontpage:

<details>
<summary>Click to expand</summary>

```rust
pub async fn do(rust_request: RustRequest) -> RustResponse {
  use crate::messages::tutorial_resource::{
    ReadRequest,
    ReadResponse,
  };

  match rust_request.operation {
    RustOperation::Create => RustResponse::default(),
    RustOperation::Read => {
      let message_bytes = rust_request.message.unwrap();
      let request_message = ReadRequest::decode(
        message_bytes.as_slice()
      ).unwrap();

      let response_message = ReadResponse {
        output_numbers: vec![1, 2, 3],
        output_string: String::from("HI"),
      };
      RustResponse {
        successful: true,
        message: Some(response_message.encode_to_vec()),
        blob: None,
      }
    }
    RustOperation::Update => RustResponse::default(),
    RustOperation::Delete => RustResponse::default(),
  }
}
```

Dart

```dart
final requestMessage = tutorialResource.ReadRequest(
  inputNumbers: [3, 4, 5],
  inputString: 'Zero-cost abstraction',
);
final rustRequest = RustRequest(
  resource: tutorialResource.ID,
  operation: RustOperation.Read,
  message: requestMessage.writeToBuffer(),
);
final rustResponse = await requestToRust(rustRequest);
```

</details>

If rinf writes a bit more code, users will only need to write the following code instead:

<details>
<summary>Click to expand</summary>

```rust
pub async fn do_read(request: ReadRequest) -> ReadResponse {
    ReadResponse {
        output_numbers: vec![1, 2, 3],
        output_string: String::from("HI"),
    }
}
```

Dart

```dart
final request = tutorialResource.ReadRequest(
  inputNumbers: [3, 4, 5],
  inputString: 'Zero-cost abstraction',
)
final response = await tutorialResourceRead(request);
```

</details>

## 3. Example comparisons between rinf and others

### Example 1: Rinf's official example

This example is copy-pasted from rinf's official tutorial, and removes some empty line, newline, etc.

#### rinf

Implement it using rinf:

<details>
<summary>Click to expand</summary>

Protobuf:

```proto
message ReadRequest {
  repeated int32 input_numbers = 1;
  string input_string = 2;
}

message ReadResponse {
  repeated int32 output_numbers = 1;
  string output_string = 2;
}
```

Rust file 1:

```rust
pub async fn handle_tutorial_resource(rust_request: RustRequest) -> RustResponse {
    match rust_request.operation {
        RustOperation::Create => RustResponse::default(),
        RustOperation::Read => {
            let message_bytes = rust_request.message.unwrap();
            let request_message = ReadRequest::decode(message_bytes.as_slice()).unwrap();
            let response_message = ReadResponse {
                output_numbers: request_message.input_numbers.into_iter().map(|x| x + 1).collect(),
                output_string: request_message.input_string.to_uppercase(),
            };
            RustResponse {
                successful: true,
                message: Some(response_message.encode_to_vec()),
                blob: None,
            }
        }
        RustOperation::Update => RustResponse::default(),
        RustOperation::Delete => RustResponse::default(),
    }
}
```

Rust file 2:

```rust
messages::tutorial_resource::ID => sample_functions::handle_tutorial_resource(rust_request).await
```

Dart:

```dart
  final request = tutorialResource.ReadRequest(
    inputNumbers: [3, 4, 5],
    inputString: 'Zero-cost abstraction',
  );
  final rustRequest = RustRequest(
    resource: tutorialResource.ID,
    operation: RustOperation.Read,
    message: request.writeToBuffer(),
  );
  final rustResponse = await requestToRust(rustRequest);
  final response = tutorialResource.ReadResponse.fromBuffer(rustResponse.message!);
  print('${response.outputNumbers} ${response.outputString}');
```

</details>

#### flutter_rust_bridge

Use flutter_rust_bridge to implement the same thing:

<details>
<summary>Click to expand</summary>

Rust:

```rust
pub struct ReadResponse {
    pub output_numbers: Vec<i32>,
    pub output_string: String,
}

pub async fn handle_tutorial_resource(input_numbers: Vec<i32>, input_string: String) -> ReadResponse {
    ReadResponse {
        output_numbers: input_numbers.into_iter().map(|x| x + 1).collect(),
        output_string: input_string.to_uppercase(),
    }
}
```

Dart:

```dart
  final response = await handleTutorialResource(
    inputNumbers: [3, 4, 5],
    inputString: 'Zero-cost abstraction',
  );
  print('${response.outputNumbers} ${response.outputString}');
```

</details>

### Example 2: Typical scenario which users ask

States is commonly used, and both users of rinf and users of bridge have asked questions about it ([1](https://web.archive.org/web/20240107060614/https://github.com/cunarist/rinf/discussions/240), [2a](https://web.archive.org/web/20240107060617/https://github.com/fzyzcjy/flutter_rust_bridge/discussions/251), [2b](https://web.archive.org/web/20240107060619/https://github.com/fzyzcjy/flutter_rust_bridge/discussions/962)). Below is a sample implementation.

In order to avoid the rinf version to be more lengthy, the rinf implementation below has some problems not handled. For example, the object is of type `int`, thus the benefits of strong type system cannot be used. As another example, users can only use one object at one time (and all other objects will be locked).

#### rinf

<details>
<summary>Click to expand</summary>

Protobuf:

```protobuf
message CreateRequest {
  string name = 1;
}

message CreateResponse {
  int32 object_id = 1;
}

message ReadRequest {
  oneof inner {
    SizeRequest size_request = 1;
    SearchRequest search_request = 2;
  }
}

message SizeRequest {
  int32 object_id = 1;
}

message SearchRequest {
  int32 object_id = 1;
  string keyword = 2;
}

message ReadResponse {
  oneof inner {
    SizeResponse size_response = 1;
    Empty search_response = 2;
  }
}

message SearchSignal {
  int32 object_id = 1;
  oneof inner {
    string value = 2;
    Empty end = 3;
  }
}

message SizeResponse {
  int32 value = 1;
}

message DeleteRequest {
  int32 object_id = 1;
}
```

Rust:

```rust
struct WordDict(HashMap<String, String>);

lazy_static! {
    static ref WORD_DICT_POOL: RwLock<HashMap<i32, WordDict>> = RwLock::new(HashMap::new());
    static ref WORD_DICT_NEXT_ID: Mutex<i32> = Mutex::new(0);
}

pub async fn handle_word_dict_resource(rust_request: RustRequest) -> RustResponse {
    match rust_request.operation {
        RustOperation::Create => {
            let request_message = CreateRequest::decode(rust_request.message.unwrap().as_slice()).unwrap();
            let object_id = {
                let mut next_id = WORD_DICT_NEXT_ID.lock().unwrap();
                *next_id += 1;
                *next_id
            };
            WORD_DICT_POOL.write().unwrap().insert(object_id, WordDict(fake_data(request_message.name)));
            let response_message = CreateResponse { object_id };
            RustResponse {
                successful: true,
                message: Some(response_message.encode_to_vec()),
                blob: None,
            }
        }
        RustOperation::Read => {
            let request_message: ReadRequest = ReadRequest::decode(rust_request.message.unwrap().as_slice()).unwrap();
            match request_message.inner.unwrap() {
                Inner::SizeRequest(request) => {
                    let pool_lock = WORD_DICT_POOL.read().unwrap();
                    let word_dict = pool_lock.get(&request.object_id).unwrap();
                    let response_message = SizeResponse { value: word_dict.0.len() as i32 };
                    RustResponse {
                        successful: true,
                        message: Some(response_message.encode_to_vec()),
                        blob: None,
                    }
                }
                Inner::SearchRequest(request) => {
                    tokio::task::spawn_blocking(move || {
                        let pool_lock = WORD_DICT_POOL.read().unwrap();
                        let word_dict = pool_lock.get(&request.object_id).unwrap();
                        for (k, v) in word_dict.0.iter() {
                            if k.contains(&request.keyword) {
                                let signal_message = SearchSignal {
                                    object_id: request.object_id,
                                    inner: Some(search_signal::Inner::Value(v.clone())),
                                };
                                send_rust_signal(RustSignal {
                                    resource: ID,
                                    message: Some(signal_message.encode_to_vec()),
                                    blob: None,
                                });
                            }
                            sleep(Duration::from_millis(500)); // Mimic slow search
                        }
                        let signal_message = SearchSignal {
                            object_id: request.object_id,
                            inner: Some(search_signal::Inner::End(Empty {})),
                        };
                        send_rust_signal(RustSignal {
                            resource: ID,
                            message: Some(signal_message.encode_to_vec()),
                            blob: None,
                        });
                    });
                    RustResponse {
                        successful: true,
                        message: None,
                        blob: None,
                    }
                }
            }
        }
        RustOperation::Update => RustResponse::default(),
        RustOperation::Delete => {
            let request_message = DeleteRequest::decode(rust_request.message.unwrap().as_slice()).unwrap();
            let removed = WORD_DICT_POOL.write().unwrap().remove(&request_message.object_id);
            assert!(removed.is_some());
            RustResponse {
                successful: true,
                message: None,
                blob: None,
            }
        }
    }
}
```

Rust file 2:

```rust
messages::word_dict_resource::ID => sample_functions::handle_word_dict_resource(rust_request).await,
```

Execute search in Dart:

```dart
  final createResponse = await requestToRust(RustRequest(
    resource: wordDictResource.ID,
    operation: RustOperation.Create,
    message: wordDictResource.CreateRequest(name: 'something').writeToBuffer(),
  ));
  final dictObjectId = wordDictResource.CreateResponse.fromBuffer(createResponse.message!).objectId;

  // Listen to result of `search`
  StreamSubscription<RustSignal>? streamSubscription;
  streamSubscription =
      rustBroadcaster.stream.where((rustSignal) => rustSignal.resource == wordDictResource.ID).listen((rustSignal) {
        final signal = wordDictResource.SearchSignal.fromBuffer(rustSignal.message!);
        if (signal.objectId != dictObjectId) {
          return;
        }
        switch (signal.whichInner()) {
          case wordDictResource.SearchSignal_Inner.value:
            print(signal.value);
          case wordDictResource.SearchSignal_Inner.end:
            streamSubscription!.cancel();
          case wordDictResource.SearchSignal_Inner.notSet:
            throw Exception();
        }
      });
  // Invoke `search`
  await requestToRust(RustRequest(
    resource: wordDictResource.ID,
    operation: RustOperation.Read,
    message: wordDictResource.ReadRequest(
      searchRequest: wordDictResource.SearchRequest(objectId: dictObjectId, keyword: 'e'),
    ).writeToBuffer(),
  ));

  await requestToRust(RustRequest(
    resource: wordDictResource.ID,
    operation: RustOperation.Delete,
    message: wordDictResource.DeleteRequest(objectId: dictObjectId).writeToBuffer(),
  ));
```

Show size in Dart:

```dart
  FutureBuilder(
    future: () async {
      final response = await requestToRust(RustRequest(
        resource: wordDictResource.ID,
        operation: RustOperation.Read,
        message: wordDictResource.ReadRequest(sizeRequest: wordDictResource.SizeRequest(objectId: dictObjectId)).writeToBuffer(),
      ));
      return wordDictResource.SizeResponse.fromBuffer(response.message!).value;
    }(),
    builder: (_, sizeSnapshot) => Text('Dict size: ${sizeSnapshot.data}'),
  ),
```

</details>

#### flutter_rust_bridge

<details>
<summary>Click to expand</summary>

Rust:

```rust
#[frb(opaque)]
pub struct WordDict(HashMap<String, String>);

impl WordDict {
    pub fn open(name: String) -> WordDict {
        WordDict(fake_data(name))
    }

    #[frb(sync, getter)]
    pub fn size(&self) -> usize {
        self.0.len()
    }

    pub fn search(&self, keyword: String, sink: StreamSink<String>) {
        for (k, v) in self.0.iter() {
            if k.contains(&keyword) {
                sink.add(v.clone());
            }
            sleep(Duration::from_millis(500)); // Mimic slow search
        }
    }
}
```

Execute search in Dart:

```dart
final dict = await WordDict.open(name: 'something');
await for (final value in dict.search(keyword: 'e')) {
  print(value);
}
dict.dispose();
```

Show size in Dart:

```dart
Text('Dict size: ${dict?.size}')
```

</details>
