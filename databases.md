# Databases

Oftentimes when creating software, it's necessary to save (or, more precisely, _persist_) some application state.

As an example, when you log into your online banking system, the system has to

1. Check that it's really you accessing the system (this is called _authentication_, and is beyond the scope of this chapter)
2. Retrieve some information from _somewhere_ and show it to the user (you).

Information that is stored and meant to be long-lived is said to be [_persisted_](<https://en.wikipedia.org/wiki/Persistence_(computer_science)>), usually on a medium that can reliably reproduce the data stored.

Some storage systems, like the filesystem, can be effective for one-off or small amounts of storage, but they fall short for larger application, for a number of reasons.

This is why most software applications, large and small, opt for storage systems that can provide

-   Reliability: The data you want is there when you need it
-   Concurrency: Imagine thousands of users accessing simultaneously.
-   Consistent: You expect the same inputs to produce the same results
-   Durable: Data should remain there even in case of a system failure (power outage or system crash)

NOTE: The above bullet points are a rewording of the [_ACID principles_](https://en.wikipedia.org/wiki/ACID), it's a set of properties often expressed and used in database design.

_Databases_ are storage mediums that can provide these properties, and much much more.

Also note that, in general, there are two large branches of database types, [SQL](https://en.wikipedia.org/wiki/SQL) and [NOSQL](https://en.wikipedia.org/wiki/NoSQL). In this chapter we will be focusing on SQL databases, using the [database/sql](https://golang.org/pkg/database/sql) package and the `postgres` driver [pq](_ "github.com/lib/pq").

There is a fair bit of CLI usage in this chapter (mainly setting up the database). For the sake of simplicity we will assume that you are running a system `ubuntu` on your machine, with `bash` installed. In the near future, look into the appendix for installation on other systems.

## A note on RDBMS choice

RDBMS (**R**elational **D**ata**B**ase **M**anagement **S**ystem) is a software program that allows users to operate on the storage engine underneath (namely, the database itself).

There are many choices and many capable systems, each one with its strenghts and weaknesses. I encourage you to do some research on the subject in case you're not familiar with the different options.

In this chapter we will be using [PostgreSQL](https://www.postgresql.org/): a mature, production ready relational database that has been proven to be extremely reliable.

The reasons for this choice, include, but are not limited to:

-   Postgres doesn't hold your hand.

    While there are GUI tools for visual exploration, Postgres comes by default with only a CLI. This makes for a better understanding of the SQL commands itself, and also makes scripting much easier (we won't be covering database scripting in this guide.)

-   It's production ready

    The default settings for `PostgreSQL` are good enough to be used in a production environment (with some caveats). Using it during development helps us close the gap between the testing, staging and production environments (also referred to as [dev/prod parity](https://www.12factor.net/dev-prod-parity)). As you will soon see, this will present a challenge during development, that, when overcome, renders your entire application more reliable (hint: integration tests).

## Getting a PostgreSQL instance running

### Docker

The easiest (and cleanest) way of getting `PostgreSQL` up and running is by using `docker`. This will create the database and user

-   [`Docker` installation instructions](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
-   See [https://hub.docker.com/\_/postgres](https://hub.docker.com/_/postgres) for more details on how to use this image.

```bash
~$ docker run \
    --name my-postgres \ # name of the instance
    -e POSTGRES_DB=bookshelf_db \ # name for the database
    -e POSTGRES_USER=bookshelf_user \ # name for the database user
    -e POSTGRES_PASSWORD=secret-password \ # database password
    -p 5432:5432 \ # map port 5432 on host to docker container's 5432
	-d \ # detach process
    postgres:11.5 # get official postgres image, version 11.5
```

You may need to run the above command with elevation (prepend it with `sudo`).

### Manual installation

Install `PostgreSQL` with the package manager

```bash
~$ sudo apt-get upgrade
~$ sudo apt-get install postgresql postgresql-contrib
```

PostgreSQL installs and initializes a database called `postgres`, and a user also called `postgres`. Since this is a system-wide install, we don't want to pollute this main database with this application's tables (`PostgreSQL` uses these to store administrative data), so we will have to create a user and a database.

Note that inside the `psql` shell, anything after a double hyphen (`--`) is considered a comment

```
~$ sudo -i -u postgres # this will switch you to the postgres user
~$ psql
psql (10.10 (Ubuntu 10.10-0ubuntu0.18.04.1))
Type "help" for help

postgres=# CREATE USER bookshelf_user WITH CREATEDB PASSWORD 'secret-password';
CREATE ROLE
postgres=# CREATE DATABASE bookshelf_db OWNER bookshelf_user;
CREATE DATABASE

postgres=# -- you can view users and databases with the commands \du and \l respectively
```

## Database migrations

[Database migrations](https://en.wikipedia.org/wiki/Schema_migration) refers to the management of incremental, reversible changes and version control to the database. This is an important part of any system that implements a database, as it allows developers to reproduce the database in a deterministic manner.

Migrations are a complicated subject, well beyond the scope of this book. If I were to oversimplify, I would classify migrations into two large subcategories:

1. Those that only modify the `schema` (the tables, indexes and configuration of the database).
2. Those that modify data and/or the `schema` (a migration that adds/removes columns or rows from a table).

We will only be dealing with type no. 1 in this chapter.

### Justification

If you are wondering "What's all this about migrations?", it's because we will need them later in this chapter when we start writing integration tests for our sample application.

### Migration tools

There are excellent, open source migration management tools out there ([exhibit 1](https://github.com/golang-migrate/migrate), [exhibit 2](https://github.com/mattes/migrate), [exhibit 3](https://github.com/go-pg/migrations)).

But we will not be using those. We will write our own tool (although simpler than those linked above) instead, using the `go` toolchain. We will also try to adhere to best practices, these being:

-   Migrations should be _ordered_, so there is a definitive order each time they are run. We will simply prepend each filename with a number, allowing enough digits for a large number of files (although realistically, migrations get "squashed" into a smaller number of files on real projects, if it makes sense to do so).
-   Migrations should be _reversible_. Applying a change to a database should simple to perform, and reversing said change should be easy as well. We will write two migration files for every change, appropriately suffixed `up` and `down`.
-   Migrations should be _idempotent_. Re-applying a change that has previously been applied should yield no effects beyond those of the initial application.

With these in mind, let's dive in...

## Project

We will initially write a (simple) tool to handle our (also simple) migrations, then we will be creating a CRUD program to interact with our spiffy, real database.

## Write the test first

```golang
// migrate_test.go
package main

import (
	"testing"
)

func TestMigrateUp(t *testing.T) {
	store, removeStore := NewStore()
	defer removeStore()

	const numberOfMigrations = 1
	err := MigrateUp(store, "migrations-directory", numberOfMigrations)
	if err != nil {
		t.Errorf("received error but didn't want one: %v", err)
	}
}
```

## Try to run the test

Fails, as expected.

```bash
# command-line-arguments [command-line-arguments.test]
.\migrate_test.go:9:10: undefined: MigrateUp
FAIL    command-line-arguments [build failed]
FAIL
```

## Write enough code to make it pass

We will take some liberties with writing code, keeping it in small batches and testing as we go.

Below is some boilerplate that will assist in creating the database connection. We need an `*sql.DB` instance that we can pass on to our tests first. As we learned in the `Dependency Injection` chapter, we should make a helper method so we can acquire an instance from anywhere in our application, while passing in the dependencies, in our case, the database connection string.

Before we continue down the happy-path, we need to make sure our `MigrateUp` function works as expected (unit-test) before we attempt to call it on the real database (integration-test).

The code directly below defines an interface, which will allow us to mock the database functionality, as well as a `NewStore` function that will simplify its creation.

```golang
// bookshelf-store.go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"os"
	"time"

	_ "github.com/lib/pq"
)

type Storer interface {
	ApplyMigration(name, stmt string) error
}

type Store struct {
	db *sql.DB
}

const (
	removeTimeout = 10 * time.Second
)

func NewStore() (*Store, func()) {
	// remember to change 'secret-password' for the password you set earlier
	const connStr = "postgres://books_user:secret-password@localhost:5432/books_db"
	// if you initialized postgres with docker, the connection string will look like this
	// const connStr = "postgres://books_user:secret-password@my-postgres:5432/books_db"
	// where 'my-postgres' is the '--name' parameter passed to the docker command

	db, err := sql.Open("postgres", connStr)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Unable to connection to database: %v\n", err)
		os.Exit(1)
	}

	// exponential backoff
	remove := func() {
		deadline := time.Now().Add(removeTimeout)
		for tries := 0; time.Now().Before(deadline); tries++ {
			err := db.Close()
			retryIn := time.Second << uint(tries)
			if err != nil {
				fmt.Fprintf(os.Stderr, "error closing connection to database, retrying in %v: %v\n", retryIn, err)
				time.Sleep(retryIn)
				continue
			}
			return
		}
		log.Fatalf("timeout of %v exceeded", removeTimeout)
	}

	return &Store{db: db}, remove
}

func (s *Store) ApplyMigration(name, stmt string) error {
	_, err := s.db.Exec(stmt)
	if err != nil {
		return err
	}
	return nil
}
```

---

### Note on exponential backoff (`remove` anonymous function)

Generally when calling external services you want to account for the possibility of failure, yet the reasons why the service could fail are numerous. The database is an external service because (generally) databases run in a client-server paradigm. In our case, the database connection could fail to close for a number of reasons. This pattern is useful to give the service some time to correct itself or reset before attempting our call again. This snippet does so by waiting on powers of 2 incrementally: 1, 2, 4, 8, 16, ..., N seconds, with a hard limit set at `removeTimeout` seconds.

This is here as a demonstration mostly, where it will prove useful is during the integration tests. The test database that we create will have be destroyed after running tests, thus, it can't have read or write operations running.

Code explained

```go
remove := func() {
	// sets the deadline to Now + removeTimeout seconds
	deadline := time.Now().Add(removeTimeout)
	// it's a normal for loop, but the failure condition has been
	// swapped by `time.Now().Before(deadline)`, which returns a boolean
	// see https://golang.org/pkg/time/#Time.Before
	for tries := 0; time.Now().Before(deadline); tries++ {
		err := db.Close()
		// as `tries` increases every loop, `retryIn` becomes a
		// time.Second` unit to the `tries` of 2
		// https://play.golang.org/p/ubyLNhxE31K has an illustrative example
		retryIn := time.Second << uint(tries)
		if err != nil {
			...
			time.Sleep(retryIn)
			continue
		}
		return
	}
	// panic and log the failure
	log.Fatalf("timeout of %v exceeded", removeTimeout)
}
```

---

We could add more methods to the `Storer` interface, but at this point, we don't need them. So we won't for now.

The `ApplyMigration` method is merely a wrapper around the `sql.DB.Exec` method, but this allows us to abstract it on our unit tests, testing that the `MigrateUp`, function does what it's intended

Here is the signature for our `MigrateUp` function, with our `Storer` interface:

```golang
MigrateUp(store Storer, dir string, num int)
```

There's a lot to our `MigrateUp` function, so let's break it down and test each step.

1. It needs to check whether the `dir` passed in exists.
2. It needs to get all the filenames inside `dir`, and allow us to iterate over them.

    2.1 It should allow for ordered iteration through the files.

3. It needs to run only `up` migrations.
4. It needs to run migrations `1` through `num`, or all of them if `num == -1`.
5. Lastly, it needs to report on the success of each migration run, if a migration fails, the entire process should be halted.

Seeing as we will have a `MigrateDown` as well, and so far they seem only to differ on step `3`, we can use a little foresight and create a utility function `migrate`, which will be used by both variants.

## Write the test first

Let's write our mock store (and test) first, so we can test the `migrate` function in isolation.

```golang
// migrate_test.go
import (
	"time"
	...
)

type migration struct {
	created time.Time
	name string
	stmt string
	called int
}

type SpyStore struct {
	migrations map[string]migration
}

func (s *SpyStore) ApplyMigration(name, stmt string) error {
	var m migration
	if mig, ok := s.migrations[name]; ok {
		m = mig
		m.called++
		return nil
	}
	m := migration{
		name: name,
		stmt: stmt,
		created: time.Now(),
	}
	m.called++
	s.migrations[name] = m
	return nil
}
func NewSpyStore() {
	return &SpyStore{map[string]migration{}}
}

```

And our migration files, which will be used in the tests

```sh
~$ mkdir migrations
```

Then create the `.sql` files inside this dir, name them `0001_create_bookshelf_table.up.sql` and `0001_create_bookshelf_table.down.sql`, with the SQL below.

```sql
-- migrations/0001_create_bookshelf_table.up.sql
BEGIN;
CREATE TABLE IF NOT EXISTS books (
	id SERIAL PRIMARY KEY,
	title VARCHAR(255) NOT NULL,
	author VARCHAR(255) NOT NULL
);
CREATE UNIQUE INDEX IF NOT EXISTS  books_id_uindex ON books (id);
COMMIT;

-- migrations/0001_create_bookshelf_table.down.sql
BEGIN;
DROP INDEX IF EXISTS books_id_uindex CASCADE;
DROP TABLE IF EXISTS books CASCADE;
COMMIT;
```

Now we're ready to address the different points, one by one.

1. It needs to check whether the `dir` passed in exists.

```golang
...
func TestMigrate(t *testing.T) {
	store := NewSpyStore()
	t.Run("error on nonexistent directory", func(t *testing.T){
		err := migrate(store, "i-do-not-exist", -1)
		if err != nil {
			t.Errorf("got an error but didn't want one: %v", err)
		}
	})

	t.Run("no error on existing directory", func(t *testing.T){
		err := migrate(store, "migrations", -1)
		if err == nil {
			t.Error("wanted an error but didn't get one")
		}
	})
}
```

## Try to run the test

As expected, it fails.

```sh
# github.com/quii/learn-go-with-tests/databases/v1 [github.com/quii/learn-go-with-tests/databases/v1.test]
.\migrate_test.go:37:10: undefined: migrate
.\migrate_test.go:44:10: undefined: migrate
FAIL    github.com/quii/learn-go-with-tests/databases/v1 [build failed]
```

## Write enough code to make it pass

```go
// bookshelf-store.go
func migrate(store Storer, dir string, num int) error {
	return nil
}
```

```sh
--- FAIL: TestMigrate (0.00s)
    --- FAIL: TestMigrate/no_error_on_existing_directory (0.00s)
        migrate_test.go:45: wanted an error but didn't get one
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v1        0.608s
```

We need an error for non-existent directories. Thankfully, this is easy with `os.Stat` and `os.IsNotExist`

```go
func migrate(store Storer, dir string, num int) error {
	if _, err := os.Stat(dir); os.IsNotExist(err) {
		return fmt.Errorf("directory %q does not exist", dir)
	}
	return nil
}
```

```sh
PASS
ok      github.com/quii/learn-go-with-tests/databases/v1        0.484s
```

On to the next point

2. It needs to get all the filenames inside `dir`, and allow us to iterate over them.

We can use [`ioutil.ReadDir`](https://golang.org/pkg/io/ioutil/#ReadDir) to implement the desired functionality. Since we would also like to prevent an empty directory from breaking our code, we should check for that as well; we can use [`ioutil.TempDir`](https://golang.org/pkg/io/ioutil/#TempDir) to create temporary directories for our tests.

## Write the test first

```go
// migrate-test.go
import (
	...
	"os"
	"io/ioutil"
)
func TestMigrate(t \*testing.T) {
	...
		t.Run("error on empty directory", func(t *testing.T) {
		// create temporary directory
		tmpdir, err := ioutil.TempDir("", "test-migrations")
		if err != nil {
			fmt.Println(err)
			t.FailNow()
		}
		defer os.RemoveAll(tmpdir) // cleanup when done

		err = migrate(store, tmpdir, -1)
		if err == nil {
			t.Error("wanted an error but didn't get one")
		}
	})

	t.Run("non-empty directory attempts to migrate", func(t *testing.T) {
		// create temporary directory
		tmpdir, err := ioutil.TempDir("", "test-migrations")
		if err != nil {
			fmt.Println(err)
			t.FailNow()
		}
		defer os.RemoveAll(tmpdir) // cleanup when done

		// create temporary files
		for _, filename := range []string{
			"01.*.up.sql",
			"01.*.down.sql",
			"02.*.up.sql",
			"02.*.down.sql",
		}{
			tmpfile, err := ioutil.TempFile(tmpdir, filename)
			if err != nil {
				fmt.Println(err)
				t.FailNow()
			}
			defer os.Remove(tmpfile.Name())

			if _, err := tmpfile.Write([]byte(filename + " SQL content")); err != nil {
				tmpfile.Close()
				fmt.Println(err)
				t.FailNow()
			}
			if err := tmpfile.Close(); err != nil {
				fmt.Println(err)
				t.FailNow()
			}
		}

		err = migrate(store, tmpdir, -1)
		if err != nil {
			t.Errorf("got an error but didn't want one: %v", err)
		}
	})
}
```

## Try to run the tests

```sh
--- FAIL: TestMigrate (0.00s)
    --- FAIL: TestMigrate/error_on_empty_directory (0.00s)
        migrate_test.go:72: wanted an error but didn't get one
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.389s
```

Only one failure, and no migration input. This is because our store is completely empty (as no code is implemented yet). Let's add a check for it before we move on.

```go
// migrate-test.go
...
func TestMigrate(t \*testing.T) {
	...
	t.Run("non-empty directory attempts to migrate", func(t *testing.T) {
		...
		if len(store.migrations) == 0 {
			t.Error("no migrations in store")
		}
		for _, m  := range store.migrations {
			want := 1
			if m.called != want {
				t.Errorf("wanted %d call got %d calls for %s migration", want,  m.called, m.name)
			}
		}
	})
}
```

```sh
--- FAIL: TestMigrate (0.03s)
    --- FAIL: TestMigrate/error_on_empty_directory (0.00s)
        migrate_test.go:72: wanted an error but didn't get one
    --- FAIL: TestMigrate/non-empty_directory_attempts_to_migrate (0.03s)
        migrate_test.go:115: no migrations in store
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.521s
```

That's better.

## Write enough code to make it pass

We finally get to use the interface! Here, we call `ApplyMigration` inside our `migrate` function

```go
// bookshelf-store.go
import (
	...
	"errors"
	"io/ioutil"
	"path/filepath"
)
func migrate(store Storer, dir string, num int) error {
	...
	files, err := ioutil.ReadDir(dir)
	if err != nil {
		return err
	}

	if len(files) == 0 {
		return errors.New("empty migration file")
	}

	for _, file := range files {
		path := filepath.Join(dir, file.Name())
		content, err := ioutil.ReadFile(path)
		if err != nil {
			fmt.Fprintf(os.Stderr, "failed to read migration file %s, %v", file.Name(), err)
			return err
		}
		err = store.ApplyMigration(file.Name(), string(content))
		if err != nil {
			return err
		}
	}
	return nil
}
```

```sh
PASS
ok      github.com/quii/learn-go-with-tests/databases/v2        0.512s
```

Just to see what the failing output would look like, change the `want` variable value to 2 (so it fails) in the `non-empty directory attempts to migrate` and run the tests.

```sh
--- FAIL: TestMigrate (0.05s)
    --- FAIL: TestMigrate/non-empty_directory_attempts_to_migrate (0.05s)
        migrate_test.go:120: wanted 2 call got 1 calls for 02.017392150.down.sql migration
        migrate_test.go:120: wanted 2 call got 1 calls for 02.540392659.up.sql migration
        migrate_test.go:120: wanted 2 call got 1 calls for 0001_create_books_table.down.sql migration
        migrate_test.go:120: wanted 2 call got 1 calls for 0001_create_books_table.up.sql migration
        migrate_test.go:120: wanted 2 call got 1 calls for 01.434163769.up.sql migration
        migrate_test.go:120: wanted 2 call got 1 calls for 01.969306692.down.sql migration
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.556s
```

Uh oh. We have several problems that this output reveals:

-   Our `migrate` function is running `up` and `down` migrations indiscriminately. We need to add a testcase so that it only runs one kind at a time.
-   It's including cases from previous tests. This is an easy fix, just need to instantiate a new `SpyStore` on each test.

## Refactor

Let's take this opportunity to clean up our tests, by making some assertions and other helper functions to make the actual tests more succint.

We'll also make some error variables to better define our expected errors.

```go
// bookshelf-store.go
...
var (
	ErrMigrationDirEmpty = errors.New("empty migration directory")
	ErrMigrationDirNoExist = errors.New("migration directory does not exist")
)
...
func migrate(store Storer, dir string, num int) error {
	if _, err := os.Stat(dir); os.IsNotExist(err) {
		return ErrMigrationDirNoExist
	}
	...
	if len(files) == 0 {
		return ErrMigrationDirEmpty
	}
	...
}
```

```go
// migrate_test.go
import (
	...
	"path/filepath"
)
...
func AssertError(t *testing.T, got, want error) {
	t.Helper()
	if got == nil {
		t.Error("wanted an error but didn't get one")
	}
	if got != want {
		t.Errorf("got %v want %v", got, want)
	}
}

func AssertNoError(t *testing.T, got error) {
	t.Helper()
	if err != nil {
		t.Errorf("got an error but didn't want one: %v", err)
	}
}

func AssertStoreMigrationCalls(t *testing.T, store *SpyStore, name string, num int) {
	t.Helper()

	m, ok := store.migrations[m]
	if !ok {
		t.Errorf("migration %q does not exist in store", name)
	}
	if m.called != num {
		t.Errorf("got %d want %d calls migration %q", num, m.called, name)
	}
}

func AssertAllStoreMigrationCalls(t *testing.T, store *SpyStore, num int, direction string) {
	t.Helper()

	for _, m  := range store.migrations {
		if !strings.HasSuffix(m.name, direction+".sql") {
			continue
		}
		AssertStoreMigrationCalls(t, store, m.name, num)
	}
}

func CreateTempDir(
	t *testing.T,
	name string,
	empty bool,
) (string, []string, func()) {
	t.Helper()

	tmpdir, err := ioutil.TempDir("", name)
	if err != nil {
		fmt.Println(err)
		os.RemoveAll(tmpdir)
		t.FailNow()
	}
	filenames := make([]string, 0)
	if !empty {
		for _, filename := range []string{
			"01.*.up.sql",
			"01.*.down.sql",
			"02.*.up.sql",
			"02.*.down.sql",
		}{
			tmpfile, err := ioutil.TempFile(tmpdir, filename)
			if err != nil {
				fmt.Println(err)
				os.Remove(tmpfile.Name())
				t.FailNow()
			}
			filenames = append(filenames, filepath.Base(tmpfile.Name()))

			if _, err := tmpfile.Write([]byte(filename + " SQL content")); err != nil {
				tmpfile.Close()
				fmt.Println(err)
				os.Remove(tmpfile.Name())
				t.FailNow()
			}
			if err := tmpfile.Close(); err != nil {
				fmt.Println(err)
				os.Remove(tmpfile.Name())
				t.FailNow()
			}
		}

	}
	cleanup := func() {
		os.RemoveAll(tmpdir)
	}
	return tmpdir, filenames, cleanup
}
```

Note that now our second test `no error on existing directory` can be removed, as an empty, existing directory raises an error as well. We've added the `direction` test as well.

```go
...
import (
	...
	"strings"
)
func TestMigrate(t *testing.T) {
	t.Run("error on nonexistent directory", func(t *testing.T) {
		store := NewSpyStore()
		err := migrate(store, "i-do-not-exist", -1)

		AssertError(t, err, ErrMigrationDirNoExist)
	})
	t.Run("error on empty directory", func(t *testing.T) {
		store := NewSpyStore()
		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", true)
		defer cleanup()

		err := migrate(store, tmpdir, -1)
		AssertError(t, err, ErrMigrationDirEmpty)
	})
	t.Run("non-empty directory attempts to migrate", func(t *testing.T) {
		store := NewSpyStore()
		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
		defer cleanup()

		err := migrate(store, tmpdir, -1, "up")
		AssertNoError(t, err)
		AssertAllStoreMigrationCalls(t, store, 1, "up")
	})
	t.Run("only apply migrations in one direction", func(t *testing.T) {
		store := NewSpyStore()
		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
		defer cleanup()

		err := migrate(store, tmpdir, -1, "up")
		AssertNoError(t, err)
		AssertAllStoreMigrationCalls(t, store, 1, "up")
		for name := range store.migrations {
			if strings.HasSuffix(name, "down.sql") {
				t.Errorf("Wrong direction migration applied: %s", name)
			}
		}
	})
}
```

```sh
# github.com/quii/learn-go-with-tests/databases/v2 [github.com/quii/learn-go-with-tests/databases/v2.test]
.\migrate_test.go:81:17: too many arguments in call to migrate
        have (*SpyStore, string, number, string)
        want (Storer, string, int)
FAIL    github.com/quii/learn-go-with-tests/databases/v2 [build failed]
```

The compiler is complaining, because migrate does not yet accept a direction. Let's `DRY` things a little preemtively this time. Add the following to `bookshelf-store.go`.

```go
// bookshelf-store.go
...
const (
	UP uint = iota
	DOWN
)
...
var (
	Directions = [...]string{UP: "up", DOWN: "down"}
)
...
```

Now you can access the directions by a very explicit `Directions[UP]` or `Directions[DOWN]`.

Change the signature of `migrate` to include a direction, and a check using [`strings.HasSuffix`](https://golang.org/pkg/strings/#HasSuffix).

```go
...
import (
	"strings"
)
...
func migrate(store Storer, dir string, num int, direction string) {
	...
	for _, file := range files {
		if !strings.HasSuffix(file.Name(), direction+".sql") {
			continue
		}
		...
	}
	...
```

```sh
ok      github.com/quii/learn-go-with-tests/databases/v2        0.437s
```

Remember to modify other occurrences of `migrate` in the test file.

Looks like we accidentally covered point no. 3:

    3. It needs to run only `up` migrations.

But we are missing the 2.1 annex: it needs to be ordered. Thanfully, we can solve this with a couple of assertions and the fact that in `go`, strings are comparable.

```go
// migrate_test.go
...
import (
	...
	"sort"
)
...
func AssertOrderAscending(t *testing.T, store *SpyStore, migrations []string) {
	t.Helper()
	for i := 0; i < len(migrations) - 1; i++ {
		m0, m1 := migrations[i], migrations[i+1]
		if m0 > m1 {
			t.Errorf("wrong migration order for asc: %q before %q)", m0, m1)
		}
	}
}

func AssertOrderDescending(t *testing.T, store *SpyStore, migrations []string) {
	t.Helper()
	for i := 0; i < len(migrations) - 1; i++ {
		m0, m1 := migrations[i], migrations[i+1]
		if m0 < m1 {
			t.Errorf("wrong migration order for desc: %q before %q)", m0, m1)
		}
	}
}
```

But we don't have a way to capture migrations _in order_, we only get them from the `SpyStore`. We need to implement a return value that captures the order. This is also, indirectly helping us with point no. 5:

5. Lastly, it needs to report on the success of each migration run, if a migration fails, the entire process should be halted.

```go
// bookshelf-store.go
...
func migrate(
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {
	if _, err := os.Stat(dir); os.IsNotExist(err) {
		return nil, ErrMigrationDirNoExist
	}

	files, err := ioutil.ReadDir(dir)
	if err != nil {
		return nil, err
	}

	if len(files) == 0 {
		return nil, ErrMigrationDirEmpty
	}

	migrations := make([]string, 0)
	for _, file := range files {
		if !strings.HasSuffix(file.Name(), direction+".sql") {
			continue
		}
		path := filepath.Join(dir, file.Name())
		content, err := ioutil.ReadFile(path)
		if err != nil {
			fmt.Fprintf(os.Stderr, "failed to read migration file %s, %v", file.Name(), err)
			return nil, err
		}
		err = store.ApplyMigration(file.Name(), string(content))
		if err != nil {
			return nil, err
		}
		migrations = append(migrations, file.Name())
	}
	return migrations, nil
}
```

Remember to change implementation of `migrate` in the test, to now return two values, discarding the first one as we don't need it.

## Write the test first

Now we can test the order of the migrations:

```go
// migrate_test.go
t.Run("up migrations should be ordered ascending", func(t *testing.T){
	store := NewSpyStore()
	tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
	defer cleanup()

	migrations, _ := migrate(store, tmpdir, -1, Directions[UP])
	AssertOrderAscending(t, store, migrations)
})

t.Run("down migrations should be ordered descending", func(t *testing.T){
	store := NewSpyStore()
	tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
	defer cleanup()

	migrations, _ := migrate(store, tmpdir, -1, Directions[DOWN])
	AssertOrderDescending(t, store, migrations)
})
```

## Try to run the test

```sh
--- FAIL: TestMigrate (0.03s)
    --- FAIL: TestMigrate/down_migrations_should_be_ordered_descending (0.00s)
        migrate_test.go:98: wrong migration order for desc: "01.993770360.down.sql" before "02.441780074.down.sql")
        migrate_test.go:98: wrong migration order for desc: "02.441780074.down.sql" before "03.668063276.down.sql")
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.624s
```

The `up` migrations are in the correct order, by grace of `ioutil.Readall`. But we should implement it explicitly, as the API for `ioutil.Readall` is not in our control, and may change and break our application.

## Write the minimal amount of code to make it pass

It's a matter of using the [`sort`](https://golang.org/pkg/sort) package to sort the `files` returned by `ioutil.ReadAll`. Specifically, [`sort.SliceStable`](https://golang.org/pkg/sort/#SliceStable). Recall that our order is implemented by the filename, we need to use `file.Name()` inside our sorting functions

```go
// bookshelf-store.go
func migrate(
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {

	...

	files, err := ioutil.ReadDir(dir)
	if err != nil {
		return nil, err
	}

	switch direction {
	case Directions[DOWN]:
		sort.SliceStable(files, func(i, j int) bool { return files[j].Name() < files[i].Name() })
	default:
		sort.SliceStable(files, func(i, j int) bool { return files[i].Name() < files[j].Name() })
	}

	...
}
```

Now our tests pass

```sh
PASS
ok      github.com/quii/learn-go-with-tests/databases/v2        0.653s
```

On to point no. 4:

4. It needs to run migrations `1` through `num`, or all of them if `num == -1`.

For `up` migrations, `1` through `num` makes sense: you will run the `num` first migrations in the dir. But what about `down` migrations? What does `num` represent in this case?

-   The migration number that it will go down to?
-   or the number of files to process before stopping?
-   What if the number of files change before applying `down` migrations?

These are all important questions, but since we are adhering to best practices, our migration files will be _idempotent_, so it does not matter if they are run repeatedly.

`num` should be the number of files to process on `down` migrations, reporting appropriately the state of the database when done.

## Write the test first

Our `CreateTempDir` function creates 3 `up` files, and 3 `down` files if the `empty` boolan param is `false`. Since our `SpyStore` is tracking how many calls each migration receives, we can check that only the first `num` have been called directly, so let's do that.

Since we have to write two tests (one for `up` and one for `down` migrations), let's our tests DRY and write an assertion function using [`reflect`](https://golang.org/pkg/reflect). We need to take care to account for migrations that may not exist in the store (with a `0` value), so we'll have to modify the `got` slice artificially.

```go
// migrate_test.go
...
import (
	...
	"reflect"
)
...
func TestMigrate(t *testing.T) {
	...
	t.Run("runs as many migrations as the num param, up", func(t *testing.T) {
		store := NewSpyStore()
		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
		defer cleanup()

		migrations, _ := migrate(store, tmpdir, 2, Directions[UP])
		AssertSliceCalls(t, store, migrations, []int{1, 1, 0})
	})

	t.Run("runs as many migrations as the num param, down", func(t *testing.T) {
		store := NewSpyStore()
		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
		defer cleanup()

		// `migrations` slice is reversed, so desired order is still (1,1,0)
		migrations, _ := migrate(store, tmpdir, 2, Directions[DOWN])
		AssertSliceCalls(t, store, migrations, []int{1, 1, 0})
	})
}
...
func AssertSliceCalls(t *testing.T, store *SpyStore, migrations []string,want []int) {
	t.Helper()
	got := make([]int, 0)
	for _, m := range migrations {
		got = append(got, store.migrations[m].called)
	}
	for len(got) < len(want) {
		got = append(got, 0)
	}
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v calls for migrations %v", got, want, migrations)
	}
}
```

## Try to run the test

We get an explicit report of what was called and shouldn't have been:

```sh
--- FAIL: TestMigrate (0.04s)
    --- FAIL: TestMigrate/runs_as_many_migrations_as_the_num_param,_up (0.02s)
        migrate_test.go:113: got [1 1 1] want [1 1 0] calls for migrations [01.813038608.up.sql 02.463630274.up.sql 03.431250500.up.sql]
    --- FAIL: TestMigrate/runs_as_many_migrations_as_the_num_param,_down (0.01s)
        migrate_test.go:123: got [1 1 1] want [1 1 0] calls for migrations [03.900955562.down.sql 02.186370488.down.sql 01.533641238.down.sql]
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.566s
```

## Write the minimal amount of code to make them pass

We'll introduce a `count` variable, increment it when the migrations are applied, and break the loop as soon as `count >= num`.

```go
// bookshelf-store.go
...
func migrate(
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {
	...
	migrations := make([]string, 0)
	count := 0
	for _, file := range files {
		if count >= num {
			break
		}
		if !strings.HasSuffix(file.Name(), direction+".sql") {
			continue
		}
		path := filepath.Join(dir, file.Name())
		content, err := ioutil.ReadFile(path)
		if err != nil {
			fmt.Fprintf(os.Stderr, "failed to read migration file %s, %v", file.Name(), err)
			return nil, err
		}
		err = store.ApplyMigration(file.Name(), string(content))
		if err != nil {
			return nil, err
		}
		migrations = append(migrations, file.Name())
		count++
	}
	return migrations, nil
}
```

## Try to run the tests

Our tests now pass

PASS
ok github.com/quii/learn-go-with-tests/databases/v2 0.475s

We're missing an explicit check for `num == -1` to run all migrations.

## Write the test first

```go
// migrate_test.go
...
	t.Run("runs all migrations if num == -1", func(t *testing.T) {
		store := NewSpyStore()
		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
		defer cleanup()

		migrations, _ := migrate(store, tmpdir, -1, Directions[UP])
		AssertSliceCalls(t, store, migrations, []int{1, 1, 1})
	})
...
```

## Try to run the test

It fails, as the `count` variable introduced before is always greater than `-1`.

```sh
--- FAIL: TestMigrate (0.04s)
    --- FAIL: TestMigrate/runs_all_migrations_if_num_==_-1 (0.00s)
        migrate_test.go:132: got [0 0 0] want [1 1 1] calls for migrations []
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        1.059s
```

## Write enough code to make it pass

It's a simple fix: prepend `num != -1` to the breaking condition.

```go
// bookshelf-store.go
func migrate(
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {
	...
	for _, file := range files {
		if num != -1 && count >= num {
			break
		}
		...
}
```

And we're back to green

```sh
PASS
ok     github.com/quii/learn-go-with-tests/databases/v2        0.943s
```

## Moving on

There's a lot of nuance to point number 5, so let's break it into simpler pieces:

5. Lastly, it needs to report on the success of each migration run, if a migration fails, the entire process should be halted.

> ... it needs to report on the success of each migration run...

Our migrate function currently writes no output. A simple `fmt.Println` should get the job done. But it complicates our testing, as the default output would be `os.Stdout`. We could use `go`'s utilites (namely, `os.Pipe`) to capture `os.Stdout`'s output, but this could lead to a race condition (`migrate` would be writing to `os.Stdout` as well as the `testing`). It can get hairy.

Since `migrate` will be an internal function (remember `MigrateUp`), we can hardcode `os.Stdout` inside the `migrate` call inside `MigrateUp`. Let's add an `io.Writer` parameter to `migrate`, which then we can use to safely inspect output.

But, as always, test first!

## Write the test first

Wouldn't it be helpful if migrate also let you know how many migrations are in total? This is the right time to implement this.

```go
// migrate_test.go
...
import (
	...
	"bytes"
)
...
	t.Run("success output is expected", func(t *testing.T){
		store := NewSpyStore()

		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", false)
		defer cleanup()

		gotBuf := &bytes.Buffer{}
		migrations, _ := migrate(gotBuf, store, tmpdir, -1, Directions[UP])
		got := gotBuf.String()

		total := len(migrations)

		wantBuf := &bytes.Buffer{}
		current := 1
		for _, m := range migrations {
			str := fmt.Sprintf("applying %d/%d: %s ...SUCCESS\n", current, total, m)
			wantBuf.WriteString(str)
			current++
		}
		want := wantBuf.String()

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})
...
```

## Try to run the test

Test fails, as expected.

```sh
# github.com/quii/learn-go-with-tests/databases/v2 [github.com/quii/learn-go-with-tests/databases/v2.test]
.\migrate_test.go:143:27: too many arguments in call to migrate
        have (*bytes.Buffer, *SpyStore, string, number, string)
        want (Storer, string, int, string)
FAIL    github.com/quii/learn-go-with-tests/databases/v2 [build failed]
```

## Write the minimal amount of code for the test to run and check the failing test output

```go
// bookshelf-store.go
...
import (
	...
	"io"
)
...
func migrate(
	out io.Writer,
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {
...
```

Run the tests again

```sh
# github.com/quii/learn-go-with-tests/databases/v2 [github.com/quii/learn-go-with-tests/databases/v2.test]
.\migrate_test.go:51:20: not enough arguments in call to migrate
        have (*SpyStore, string, number, string)
        want (io.Writer, Storer, string, int, string)
.\migrate_test.go:61:20: not enough arguments in call to migrate
        have (*SpyStore, string, number, string)
        want (io.Writer, Storer, string, int, string)
.\migrate_test.go:70:20: not enough arguments in call to migrate
        have (*SpyStore, string, number, string)
        want (io.Writer, Storer, string, int, string)
...
FAIL    github.com/quii/learn-go-with-tests/databases/v2 [build failed]
```

Whoops, looks like we forgot to modify current calls to `migrate` in our tests. Let's create a `dummyWriter` variable and insert it on all but the last test we wrote.

```go
// migrate_test.go
...
...
migrate(dummyWriter, store, "i-do-not-exist", -1, Directions[UP])
...
var dummyWriter = &bytes.Buffer{}
```

Now our tests run, and yield the expected failure

```sh
--- FAIL: TestMigrate (0.08s)
    --- FAIL: TestMigrate/success_output_is_expected (0.01s)
        migrate_test.go:158: got "" want "applying 1/3: 01.734206635.up.sql ...SUCCESS\napplying 2/3: 02.168107541.up.sql ...SUCCESS\napplying 3/3: 03.660970255.up.sql ...SUCCESS\n"
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.934s
```

## Write enough code to make it pass

If `store.ApplyMigration` doesn't return an error, we can assume it was successful. This is where we'll add our message.

```go
// bookshelf-store.go
...
func migrate(
	out io.Writer,
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {
	...

	total := len(files)
	if total == 0 {
		return nil, ErrMigrationDirEmpty
	}

	migrations := make([]string, 0)
	count := 0
	for _, file := range files {
		...

		fmt.Fprintf(out, "applying %d/%d: %s ", count+1, total, file.Name())
		err = store.ApplyMigration(file.Name(), string(content))
		if err != nil {
			return nil, err
		}
		fmt.Fprint(out, "...SUCCESS\n")
		migrations = append(migrations, file.Name())
		count++
	}
	return migrations, nil
}
```

## Try to run the test

Success! oh wait...

```sh
--- FAIL: TestMigrate (0.11s)
    --- FAIL: TestMigrate/success_output_is_expected (0.00s)
        migrate_test.go:158: got "applying 1/6: 01.414028183.up.sql ...SUCCESS\napplying 2/6: 02.949529057.up.sql ...SUCCESS\napplying 3/6: 03.887873211.up.sql ...SUCCESS\n" want "applying 1/3: 01.414028183.up.sql ...SUCCESS\napplying 2/3: 02.949529057.up.sql ...SUCCESS\napplying 3/3: 03.887873211.up.sql ...SUCCESS\n"
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.807s
```

## Write enough code to make it pass

It's reporting `n/6`, it should be `n/3`. Filtering inside the main loop with `strings.HasSuffix(file.Name(), direction+".sql")` is not working for us anymore.

Let's fix that by filtering the `files` outside of the main loop.

```go
// bookshelf-store.go
...
func migrate(
	out io.Writer,
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {
	...
	allFiles, err := ioutil.ReadDir(dir)
	if err != nil {
		return nil, err
	}
	files := make([]os.FileInfo, 0)
	for _, f := range allFiles {
		if strings.HasSuffix(f.Name(), direction+".sql") {
			files = append(files, f)
		}
	}

	for _, file := range files {
		if num != -1 && count >= num {
			break
		}
		path := filepath.Join(dir, file.Name())
		...
	}
	return migrations, nil
}
```

Now our tests are passing.

```sh
PASS
ok      github.com/quii/learn-go-with-tests/databases/v2        1.129s
```

## On to 5-2

5. Lastly, it needs to report on the success of each migration run, if a migration fails, the entire process should be halted.

> ... if a migration fails, the entire process should be halted.

This already happens, as we return the error if it fails. But how do we _test_ this?

This is prime material for the integration tests, as the database engine itself will tell you if the `sql` inside the migration file is bad or cannot be executed. And we will get to this.

We need to simulate a failure. We can do this within our `SpyStore`.

Given that we will test this anyway (via the integration tests), we don't have to parse any `sql` inside the migration files; this is a whole different beast that we fortunately don't have to deal with.

Since we used an interface (`Storer`) to abstract implementation, we can put whatever we want inside `SpyStore.ApplyMigration` for the failure condition.

Pies vs Cakes is a very serious debate that has raged on since forever. Pie is clearly superior. So we decided to forbid any cake-related SQL. Our failure condition will be the word `cake` inside the `sql` files.

Let's move on to our tests

## Write the test first

The failure should be reported with a helpful error message as well, so as not to leave our users helplessly looking through their files.

We will ignore errors for the sake of brevity.

```go
// migrate_test.go
...
import (
	...
	"errors"
)
...
var errNoCakeSQL = errors.New("cakeSQL is not allowed")
...
func (s *SpyStore) ApplyMigration(name, stmt string) error {
	if strings.Contains(strings.Lower(stmt), "cake") {
		return errNoCakeSQL
	}
	// the rest of the method is unchanged
	...
}
...
	t.Run("failure output is expected", func(t *testing.T){
		store := NewSpyStore()
		tmpdir, _, cleanup := CreateTempDir(t, "test-migrations", true)
		defer cleanup()

		tmpfile, _ := ioutil.TempFile(tmpdir, "01.cake.*.up.sql")
		tmpfile.Write([]byte("cake is superior! end pie tyranny")); err != nil {
		tmpfile.Close()

		gotBuf := &bytes.Buffer{}
		_, err = migrate(gotBuf, store, tmpdir, -1, Directions[UP])
		got := gotBuf.String()

		wantBuf := &bytes.Buffer{}
		str := fmt.Sprintf(
			"applying 1/1: %s ...FAILURE: %v\n",
			filepath.Base(tmpfile.Name()),
			errNoCakeSQL,
		)
		wantBuf.WriteString(str)
		want := wantBuf.String()

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
		AssertError(t, err, errNoCakeSQL)
	})
```

## Try to run the test

As expected, since there is no code for the failure report yet.

```sh
--- FAIL: TestMigrate (0.07s)
    --- FAIL: TestMigrate/failure_output_is_expected (0.00s)
        migrate_test.go:191: got "applying 1/1: 01.cake.702313345.up.sql " want "applying 1/1: 01.cake.702313345.up.sql ...FAILURE: cakeSQL is not allowed\n"
FAIL
exit status 1
FAIL    github.com/quii/learn-go-with-tests/databases/v2        0.647s
```

## Write enough code to make it pass

```go
// bookshelf-store.go
...
func migrate(
	out io.Writer,
	store Storer,
	dir string,
	num int,
	direction string,
) ([]string, error) {
	...
	for _, file := range files {
		...
		err = store.ApplyMigration(file.Name(), string(content))
		if err != nil {
			fmt.Fprintf(out, "...FAILURE: %v\n", err)
			return nil, err
		}
		...
	}
	return migrations, nil
}
```

And we are green again

```sh
PASS
ok      github.com/quii/learn-go-with-tests/databases/v2        1.555s
```
