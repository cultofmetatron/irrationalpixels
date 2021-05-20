---
title: "Database Modeling with Ecto: Part 1 - CRUD operations"
date: 2021-04-03T19:50:33-04:00
draft: false
---

## Creating your first records

# Contents

1. **Relational modelling in Ecto**
1. **Ecto Schema Fundamentals**

# Relational modelling in Ecto

You've heard of Elixir and are just getting started with your first taste of the kool-aid.

If you're new to Elixir, you're probably finding Ecto to be a bit of a culture shock.
Ecto is an immensly powerful library for working with SQL, but unlike other ORMs you may have used, Ecto embraces what makes SQL great.
ORMs like Active Record or Django's Python objects try to hide SQL behind their language specifc interfaces.
SQL Land however, does not deal with objects.
Ecto takes SQL as it is and gives you very powerful tools for modeling your application's database layer without building abstractions that ultimately hinder you later on with slow queries and magic behavior that gets in your way.

Of course, this means you need to know how to think in SQL and the relational model.

### Thinking in Relationships

I'll spare you the theoretical jibber jabber about industry jargon like "third normal form" and "denormalization".
Frankly, a lot of books on SQL modeling get high on the theoretical horse to cover every single possible edge case imaginable.
As somone who learned SQL on the job, I've distilled it down to a a few rules of thumb with the occasional exception.

The secret to thinking in SQL is right here in the word "relational." Almost all relations you will model fit one of three categories.

1. A `has many` B
2. B `belongs to` A
3. A `has many` C and B `has many` C
4. A `has many` B `through` C (if B also `has_many` A `through` C, we might call this a `many to many` relationship).

Any nontrivial database schema is going to have all of these relationships modeled in one way or another.
To make this all a bit more relatable, let's plan a schema for a school grade tracker.
This could very well be the heart of a system that helps teachers quickly grade their students and send out report cards.

Consider the following relationships:

- A `Class` has one `Teacher`: `belongs to`
- A `Student` attends many `Class`: `has many`
- A `Class` has many `Student`: `has many`
- A `Class` has many `Student` and only one `Teacher`: `many to many`

Right here we have a few relationships mapped out.
Given the following the information, we can also infer some secondary effect:

- a `Student` has many `Teacher` `through` a `Class`
- a `Teacher` has many `Student` `through` a `Class`

We still havent gotten to grades! But where to store it?
Thinking about grades, I would model it as an operation over a group of `Assignment`.
It makes sense to think about it in terms of the `Enrollment` of the students.

With that we can add a few more relationships and associated constraints.

- An `Enrollment` `joins` a `Student` and `Class` to establish a `many to many` relationship.
- An `Enrollment` `has many` `Assignment`

Now that we have a birds eye map of the schema we want to build and model, we can start to hash out the details in code!

### Ecto Schema Fundamentals

First, install the Phoenix generator and run the following to create your project:

```zsh
mix phx.new --binary-id --database postgres grade_tracker
```

Every record in yoru database needs a `primary key`.
This is a column that serves as a unique key for a particular record in a table.
By default, Phoenix sets up autoincrementing integers as the primary keys.
Setting the `--binary-id` flag ensures we'll be generating uuid primary keys
You are more than welcome to use the default autoincrement integers instead.
They are faster and arguably easier to index.
However, uuids are harder to blindly guess and will be useful if later you decide to spread your data out to multiple systems.

We need a teachers table and schema so let's fire up the generator

```zsh
$ mix phx.gen.schema Teacher teachers name:string
* creating lib/grade_tracker/teacher.ex
* creating priv/repo/migrations/20210414205427_create_teachers.exs
```

This generates a migration that creates our `teachers` table and a schema file.

```elixir
# priv/repo/migrations/20210414205427_create_teachers.exs
defmodule GradeTracker.Repo.Migrations.CreateTeachers do
  use Ecto.Migration

  def change do
    create table(:teachers, primary_key: false) do
      add :id, :binary_id, primary_key: true
      add :name, :string

      timestamps()
    end

  end
end

```

By default, we get 4 colums created. `id` is set as the primary key, and `name` completes the list of ones we created.
`timestamps()` adds `inserted_at` and `updated_at` columns.

The schema that was generated is the interface for how we will interact with this table.

```elixir
defmodule GradeTracker.Teacher do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  schema "teachers" do
    field :name, :string

    timestamps()
  end

  @doc false
  def changeset(teacher, attrs) do
    teacher
    |> cast(attrs, [:name]) # pulls the name attribute into the changeset
    |> validate_required([:name]) # validates that a name attribute exists either on the teacher or the attrs
  end
end
```

Instead of objects, Elixir gives us [structs](https://elixircasts.io/intro-to-structs).
The distinction is that there are no instance methods or mutable state.
It represents the state of the record as it was retrieved from the backend.
A struct is a data structure similar to [Map](https://hexdocs.pm/elixir/1.0.5/Map.html) with a difference that the keys are defined up front.

In this module, `use Ecto.Schema` means that the `Teacher` module inherits the behavior of the an Ecto Schema.
It does this through a macro system which is beyond the scope of this article but worth delving into.

As you can see, there is one function defined.
The `changeset/2` function takes a schema struct and a set of params.
It returns an Ecto.Changeset record that carries with it information pertaining to a database change oepration.
Ecto expects that you may have different operations with different validation rules depending on the context of the operation, and it's perfectly fine to have different functions for generating different types of changesets.

> For a complete list of built in validations, [checkout the docs for Ecto.Changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html)

### Inserting records

Database Operations are done via `Repo` aliased from `GradeTracker.Repo`. It's a microservice that you pass data to and get back the result of your operations.
To insert a record, we create a `Teacher` changeset.
Since we are creating a new one, we can pass in a new Teacher struct along with some paramaters.

Lets test out our new teacher schema. Create a file `test/schemas/teacher_test.exs`

```elixir

defmodule GradeTracker.TeacherSchemaTest do
  use  GradeTracker.DataCase
  alias GradeTracker.Repo
  alias GradeTracker.Teacher

  test "we can insert a teacher", _ do
    assert {:ok, %Teacher{ id: _id, name: "Jose Valim" } = _teacher} = %Teacher{}
    |> Teacher.changeset(%{
      name: "Jose Valim"
    })
    |> Repo.insert()

  end
end
```

Upon a successsful insertion, the insert operation returns `{:ok, *struct* }`.
The struct returned represents the state of that record in the database.
Not only does it have a name attribute, it also has an id which was generated during the insertion.

Now let's run this test!

```bash
$ mix test
....

Finished in 0.1 seconds
4 tests, 0 failures

Randomized with seed 80769
```

For completeness sake, let's verify that an error gets returned if we leave off the `:name` attr.
If `Repo.insert/1` errors, we get back an error tuple giving us a changeset with the errors embedded within.

```elixir
defmodule GradeTracker.TeacherSchemaTest do
  #...

  test "inserting a teacher errors without a name", _ do
    assert {:error, %Ecto.Changeset{
      errors: [name: {"can't be blank", [validation: :required]}],
      valid?: false
    } = _changeset} = %Teacher{}
    |> Teacher.changeset(%{})
    |> Repo.insert()
  end

end
```

### Updating records

`Repo.update/1` accepts a changeset and operates similarly to `Repo.insert/1` with the exception that a primary key must exist on the struct passed into the changeset creation.
If you try to update a record without a primary key, you will raise an exception rather than get an error tuple.

```elixir
defmodule GradeTracker.TeacherSchemaTest do
  #...

  test "updating a teacher errors without an id", _ do
    assert_raise Ecto.NoPrimaryKeyValueError, fn ->
      %Teacher{}
      |> Teacher.changeset(%{ name: "Dave Thomas" })
      |> Repo.update()
    end
  end
end
```

### Deleting a record

`Repo.delete/1` accepts a struct and deletes it. You'll get back `{:ok, teacher}` that returns the deleted record

```elixir
  test "deleting a teacher", _ do
    assert {:ok, teacher} = %Teacher{}
    |> Teacher.changeset(%{
      name: "Jose Valim"
    })
    |> Repo.insert()

    assert {:ok, teacher} = teacher
    |> Repo.delete()


  end
```

And with that, you can now create, update and delete a single record. In my followup, I'll cover how to setup relations and transactions.
