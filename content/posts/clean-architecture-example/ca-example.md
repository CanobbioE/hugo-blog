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
However, having the database behind a `FileProvider` abstraction, allows us to easily swap the underlying implementation in case we wanted to switch to a third party provider.

### Provider
The `FileProvider` has to know about the most external layer ([image for reference](/posts/cleaner-architecture/CleanArchitecture.jpeg)). Therefore, its implementation should be under the `infrastructure` folder.

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

This works in a single-service environment, in case of multiple services withing a single project, the database implementation should be moved to a shared `pkg/` folder, to reduce code's redundancy.

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
Data manipulation happens in the `usecase` folder. The entrypoint of any usecase is a controller.
A **controller** receives data from the outmost layer and prepares it in a way the **interactor** can g
