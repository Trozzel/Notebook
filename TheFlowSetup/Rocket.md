# Working with Rocket

## Notes

### ToDo App Tutorial Link

[Build a todo app with Rust and rocket](https://codevoweb.com/build-a-simple-api-with-rust-and-rocket/)

#### Synopsis

He installed a tool that allows for dynamic updating of webpage:

```cargo
cargo install cargo-watch
```

Then, to start the server:

```bash
cargo watch -q -c -w src/ -x run
```

##### Structure of Project

1.  **Models**

    A **model** is the struct that will represent your data. In this example, it represents a ToDo item.

    This will `#[derive(Serialize, Deserialize, Clone)]` for interoperability with `Json` and for easy Cloning

    In this example, in order to have a container of `Todo` he has a struct with one element:
    > `Arc<Mutex<Vec<Todo>>>`

    ```rust
    // src/model.rs
    #allow(non_snake_case)
    #derive(Serialize, Deserialize, Debug, Clone)
    pub struct Todo {
        pub id: Option<String>,
        pub title: String,
        pub content: String,
        pub completed: Option<bool>,
        pub createdAt: Option<DateTime<Utc>>,
        pub modifiedAt: Option<DateTime<Utc>>,
    }

    pub struct AppState {
        pub todo_db: Arc<Mutex<Vec<Todo>>>,
    }

    impl AppState {
        pub fn init() -> AppState {
            AppState {
                todo_db: Arc::new(Mutex::new(Vec::new())),
            }
        }
    }

    // Unique Purpose Struct
    /******************************************************************************/
    // The following struct is used for PATCH requests and is used as the data
    // sent from the client to the server used to update existing data on server
    #[allow(non_snake_case)]
    #[derive(Debug, Deserialize)]
    pub struct UpdateTodoSchema {
        pub title: Option<String>,
        pub content: Option<String>,
        pub completed: Option<bool>,
    }
    ```



2.  **Responses**

    A **Response** is what the rocket handler returns, wrapped in a `Result<Json<**Response Type**>, Status>`

    You can have a `GenericResponse` with a `status` and a `message` field. You can use this for any generic message you may want to return.

    ```rust
    // src/response.rs
    #[derive(Serialize)]
    pub struct GenericResponse {
        pub status: String,
        pub message: String,
    }
    ```

    ❓ So, how do you give a custom object (i.e. a `Todo`) as a `Response`?

    1.  You wrap you object in a `struct`

        ```rust
        // src/response.rs
        #[derive(Serialize, Debug)]
        pub struct TodoData {
            pub todo: Todo,
        }
        ```

    2.  You place that `struct` (i.e. the `TodoData`) into a Response `struct`

        ```rust
        // src/response.rs
        #[derive(Serialize, Debug)]
        pub struct SingleTodoResponse {
            pub status: String,
            pub data: TodoData,
        }
        ```

        _Question_ - Why not have `data: Todo`, since `Todo` is `Serialize, Deserialize`?

    3.  To create a `Response` for a container, add a field for the amount being returned

        ```rust
        // src/response.rs
        pub struct TodoListResponse {
            pub status: String,
            pub results: usize,
            pub todos: Vec<Todo>,
        }
        ```

3.  **Handlers**

    A **handler** the function that handles the request

    - 📝 **NOTES:**
        - All `Ok` trait portion of `Result` trait return type is `Json<`_CustomResponse_`>`
        - All `Error` trait portion of `Result` trait return type is `Custom<Json<GenericResponse>>`
        - _Example_: ` -> Result<Json<SingleTodoResponse>, Custom<Json<GenericResponse>>>`

    1. Get a list of todo items

       ```rust
       // src/handler.rs
       #[get("/todos?<page>&<limit>")]
       pub async fn todos_list_handler(
           page: Option<usize>,
           limit: Option<usize>,
           data: &State<AppState>,
       ) -> Result<Json<TodoListResponse>, Status> {
           let vec = data.todo_db.lock().unwrap();

           let limit = limit.unwrap_or(10);
           let offset = (page.unwrap_or(1) - 1) * limit;

           let todos: Vec<Todo> = vec.clone().into_iter().skip(offset).take(limit).collect();

           let json_response = TodoListResponse {
               status: "success".to_string(),
               results: todos.len(),
               todos,
           };
           Ok(Json(json_response))
       }
       ```

       **Explanation:**

       - `data: &State<AppState>`
         - `State` is from `rocket::State`
           - This must be a way to work with persistent data
         - `AppState` is a custom struct created earlier as a container of `Todo`'s
         - _This is not provided by client in the request to the server_
       - Place the vector into the appropriate response (i.e. `TodoListReponse`)
       - Returns `Ok<Json<TodoListResponse>>`
       - ❓ I don't understand what return of `Status` is and how could return `Error<Status>`❓

    2. Adding a record, via POST
    
       📝 NOTES: 
        - `mut body: Json<Todo>` automatically handles verification of POST request body
        - You can access fields of `Json` serialized objects (i.e. `body.title`)

       ```rust
       // src/handles.rs
       #[post("/todos", data = "<body>")]
        pub async fn create_todo_handler(
            mut body: Json<Todo>,
            data: &State<AppState>,
        ) -> Result<Json<SingleTodoResponse>, Custom<Json<GenericResponse>>> {

            // If todo with same name exists, then ...
            {
                ...
                return Err(Custom(Status::Conflict, Json(GenericResponse{status:..., message:...})));
            }

            // Else, add todo to datastore (i.e. `Arc<Mutex<Vec<Todo>>>`)

            // Finally, return success response
            let json_response = SingleTodoResponse {
                status: "success".to_string(),
                data: TodoData {
                    todo: todo.into_inner(),
                },
            };

            Ok(Json(json_response))
       ```
       where:
        - `Custom`:
            - `rocket::response::status::Custom`
        - `Status`:
            - `rocket::http::Status`
    
    3. Get single todo, by id, via GET 

       ```rust
       // src/handler.rs
       #[get("/todos/<id>")]
        pub async fn get_todo_handler(
            id: String,
            data: &State<AppState>,
        ) -> Result<Json<SingleTodoResponse>, Custom<Json<GenericResponse>>> {
            let vec = data.todo_db.lock().unwrap();

            for todo in vec.iter() {
                if todo.id == Some(id.to_owned()) {
                    let json_response = SingleTodoResponse {
                        status: "success".to_string(),
                        data: TodoData { todo: todo.clone() },
                    };

                    return Ok(Json(json_response));
                }
            }

            let error_response = GenericResponse {
                status: "fail".to_string(),
                message: format!("Todo with ID: {} not found", id),
            };
            Err(Custom(Status::NotFound, Json(error_response)))
        }
       ```

    4. Edit a todo record, by id, using PATCH
       
       ```rust
       #[patch("/todos/<id>", data = "<body>")]
        pub async fn edit_todo_handler(
            id: String,
            body: Json<UpdateTodoSchema>,
            data: &State<AppState>,
        ) -> Result<Json<SingleTodoResponse>, Custom<Json<GenericResponse>>> {
            let mut vec = data.todo_db.lock().unwrap();

            for todo in vec.iter_mut() {
                if todo.id == Some(id.clone()) {
                    let datetime = Utc::now();
                    let title = body.title.to_owned().unwrap_or(todo.title.to_owned());
                    let content = body.content.to_owned().unwrap_or(todo.content.to_owned());
                    let payload = Todo {
                        id: todo.id.to_owned(),
                        title: if !title.is_empty() {
                            title
                        } else {
                            todo.title.to_owned()
                        },
                        content: if !content.is_empty() {
                            content
                        } else {
                            todo.content.to_owned()
                        },
                        completed: if body.completed.is_some() {
                            body.completed
                        } else {
                            todo.completed
                        },
                        createdAt: todo.createdAt,
                        updatedAt: Some(datetime),
                    };
                    *todo = payload;

                    let json_response = SingleTodoResponse {
                        status: "success".to_string(),
                        data: TodoData { todo: todo.clone() },
                    };
                    return Ok(Json(json_response));
                }
            }

            let error_response = GenericResponse {
                status: "fail".to_string(),
                message: format!("Todo with ID: {} not found", id),
            };

            Err(Custom(Status::NotFound, Json(error_response)))
       }
       ```
         - Summary:
           - Receives an `id` and a `UpdateTodoSchema` (`title: Option<String>`, `content: Option<String>`, `completed: Option<String>`), as a request.
           - Iterates through Todo's to find matching `id`
           - Then, updates any fields that were sent as part of the `UpdateTodoSchema`
           - Finally, returns `SingleTodoResponse`
    
    5.  Deleting a todo item with DELETE
        ```rust
        #[delete("/todos/<id>")]
        pub async fn delete_todo_handler(
            id: String,
            data: &State<AppState>,
        ) -> Result<Status, Custom<Json<GenericResponse>>> {
            let mut vec = data.todo_db.lock().unwrap();

            for todo in vec.iter_mut() {
                if todo.id == Some(id.clone()) {
                    vec.retain(|todo| todo.id != Some(id.to_owned()));
                    return Ok(Status::NoContent);
                }
            }

            let error_response = GenericResponse {
                status: "fail".to_string(),
                message: format!("Todo with ID: {} not found", id),
            };
            Err(Custom(Status::NotFound, Json(error_response)))
        }
        ```

4.  **Finally, create the routes in** `main.src`

    ```rust
    use handler::{
    create_todo_handler, delete_todo_handler, edit_todo_handler, get_todo_handler,
    health_checker_handler, todos_list_handler,
    };

    #[macro_use]
    extern crate rocket;

    mod handler;
    mod model;
    mod response;

    #[launch]
    fn rocket() -> _ {
        let app_data = model::AppState::init();
        rocket::build().manage(app_data).mount(
            "/api",
            routes![
                health_checker_handler,
                todos_list_handler,
                create_todo_handler,
                get_todo_handler,
                edit_todo_handler,
                delete_todo_handler
            ],
        )
    }
    ```

### Pretty cool!
