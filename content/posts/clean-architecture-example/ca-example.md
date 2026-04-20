---
title: "A Clean Architecture Example -- aka how I (try to) organize my projects"
date: 2022-06-20T17:13:11+01:00
showDate: false
draft: true
tags: ["blog","architecture","programming","opinions", "go", "golang"]
---

## Intro
If you have read my previous post about [Clean Architecture Variants](/posts/cleaner-architecture/cleaner-architecture), you might be curious about how I use the paradigm in my projects.

If you haven't... well you should read it first.

## Let's Code
I usually enjoy coding in Golang, therefore, that's what we are going to use today.
For this example, we want to create a simple storage service: a user can **store**, **retrieve** or **delete** a file.

First thing first, the folder structure:

```
filestorage
|_ domain
|_ infrastructure
|  |_ db
|  |_ provider
|  |_ webservice
|_ interface
|  |_ controller
|  |_ presenter
|  |_ repository
|_ usecase
   |_ interactor
```

### Domain
The domain defines the core entities. In our example, we'll need just a `File`.

```golang
// filestorage/domain/file.go

type File struct {
    Name    string
    Content string
}
```

### Repository
A repository is an adapter between the domain's entity and the actual storage, which could be either a database, an external provider, a mocked service or just a simple file.
Whichever it is, the repository's implementation should be under the `interface/repository` folder.

```golang
// filestorage/interface/repository/file_repository.go

type FileRepository interface {
	Store(context.Context, *domain.File) (*domain.File, error)
	Get(context.Context, *domain.File) (*domain.File, error)
	Delete(context.Context, *domain.File) (*domain.File, error)
}

type Files struct {
	provider provider.FileProvider
}

func (fr *Files) Store(ctx context.Context, f *domain.File) (*domain.File, error) {
	providerFile, err := fr.provider.Create(ctx, fc.toProvider(f))
	if err != nil {
		return nil, err
	}

	return fr.fromProvider(providerFile)
}

// implement Get and Delete...
```

As you might have noticed, the `FileRepository` implementation relies on the `FileProvider` interface.
Since we are relying on a database to store our files, the code will, eventually, call the database.  
However, having the database behind a `FileProvider` abstraction allows us to easily swap the underlying implementation in case we wanted to switch to a third party provider.

### Provider
The `FileProvider` has to know about the most external layer (see the Clean Architecture diagram above). Therefore, its implementation should be under the `infrastructure` folder.

```golang
// filestorage/infrastructure/provider/db_file_provider.go
type File struct {
    // ...provider specific fields...
}

type FileProvider interface {
	Get(context.Context, *File) (*File, error)
    Create(context.Context, *File) (*File, error)
	Delete(context.Context, *File) (*File, error)
}

type DbFileProvider struct {
	db db.Handler
}

// implement Get, Create and Delete
```

### Database
The `db.Handler` is implemented in the `infrastructure/db` folder.

The implementation's specifics depend on which database we want to use, but the `DbHandler` interface should be implemented by any DB driver.

This works in a single-service environment. In case of multiple services within a single project, the database implementation should be moved to a shared `pkg/` folder to reduce code redundancy.

```golang
// filestorage/infrastructure/db/sql.go

type Handler interface {
	Save(context.Context, interface{}) (interface{}, error)
	First(context.Context, interface{}) (interface{}, error)
	Find(context.Context, interface{}) (interface{}, error)
	Delete(context.Context, interface{}) (interface{}, error)
}

type SQLHandler struct {
	Conn   lib.DB
	Config *SQLConfig
}

// implement Save, First, Find, Delete
```

Now that the code to internally manage the domain's entities is in place, we can focus on defining the business logic.

### Controller
Data manipulation happens in the `usecase` folder. The entry point of any use case is a controller.
A **controller** receives data from the outermost layer and prepares it in a way the **interactor** can process.

```golang
// filestorage/interface/controller/file_controller.go

type FileController struct {
	Interactor usecase.FileUseCase
}

func (fc *FileController) Store(ctx context.Context, file *domain.File) (*domain.File, error) {
	return fc.Interactor.StoreFile(ctx, file)
}

func (fc *FileController) Get(ctx context.Context, file *domain.File) (*domain.File, error) {
	return fc.Interactor.GetFile(ctx, file)
}

func (fc *FileController) Delete(ctx context.Context, file *domain.File) (*domain.File, error) {
	return fc.Interactor.DeleteFile(ctx, file)
}
```

### Interactor
The interactor contains the actual business logic.  
It coordinates between the repository and any other component, and it implements the application's use cases.

```golang
// filestorage/usecase/interactor/file_usecase.go

type FileUseCase interface {
	StoreFile(context.Context, *domain.File) (*domain.File, error)
	GetFile(context.Context, *domain.File) (*domain.File, error)
	DeleteFile(context.Context, *domain.File) (*domain.File, error)
}

type FileInteractor struct {
	Repo interface_repository.FileRepository
}

func (fi *FileInteractor) StoreFile(ctx context.Context, file *domain.File) (*domain.File, error) {
	return fi.Repo.Store(ctx, file)
}

func (fi *FileInteractor) GetFile(ctx context.Context, file *domain.File) (*domain.File, error) {
	return fi.Repo.Get(ctx, file)
}

func (fi *FileInteractor) DeleteFile(ctx context.Context, file *domain.File) (*domain.File, error) {
	return fi.Repo.Delete(ctx, file)
}
```

### Presenter
The presenter formats the output from the interactor into a structure suitable for the external interface.  
For example, it can convert data into a JSON response for an HTTP API.

```golang
// filestorage/interface/presenter/file_presenter.go

type FilePresenter struct{}

func (fp *FilePresenter) Present(file *domain.File) map[string]interface{} {
	return map[string]interface{}{
		"name":    file.Name,
		"content": file.Content,
	}
}
```

## Wrapping up
If you look at the architecture diagram again, you'll see how everything fits:

- **Entities** live in `domain`
- **Use Cases** are implemented in `usecase/interactor`
- **Interface Adapters** are our `controller`, `presenter`, and `repository`
- **Frameworks and Drivers** live in `infrastructure`

The flow of control always goes from the outer circles to the inner circles. Dependencies point inward.  
This structure allows you to:

- change frameworks without affecting business rules
- replace the database without rewriting business logic
- write proper unit tests with no reliance on external systems

Of course, this is a simplified example. In a real system, you would have more layers, error handling, logging, and likely many more abstractions.  
But the core idea remains the same: isolate the business rules from the external world.

If you enjoyed this post, let me know, and I can write a more advanced example next time.
