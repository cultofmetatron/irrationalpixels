---
title: "Ecto up and running"
date: 2021-04-03T19:50:33-04:00
draft: true
---

So you just started working with phoenix and you need to learn to use the sql layer.
Ecto is an immensly powerful library for working with sql.
Unlike other ORMs you may have used, Ecto embraces what makes sql great and manages to add to that rather trying to make it something else entirely.

The purpose of this guide is to get you up to speed with designing a Database schema and interacting with it from Ecto.
You may have used other orms in the past but I'll assume you dont' know sql.


Lets create an app that allows tracking grades for teachers.
We weant to track a set of teachers, students and classes.
So lets think about some considerations we want to map out.
There are many constraints asociated with the schema we want to model and it helps to list them all out.


* A `Class` has one `Teacher`: `belongs to`
* A `Student` attends many `Class`: `has many`
* A `Class` has many `Student`: `has many`
* A `Class` has many `Student` and only one `Teacher`:  `many to many`

Right here we have a few relationships mapped out.
Given the following the information, we can also infer some secondary effect

* a `Student` has many `Teacher` `through` a `Class`
* a `Teacher` has many `Student` `through` a `Class`

We still havent gotten to grades! But where to store it?
Thinking about grades, I would model it as an operation over a group of `Assignment`.
It makes sense to think about it in terms of the `Enrollment` of the students.

With that we can add a few more relationships and associated constrants

* An `Enrollment` `joins` a `Student` and `Class` to establish a `many to many` relationship
* An `Enrollment` `has many` `Assignment`



By default, phoenix sets up autoincrementing integers as the primary keys.
Setting the `--binary-id` flag ensures we'll be generating uuid primary keys 
You are more than welcome to use the default autoincrement integers instead. 
They are faster and arguably easier to index.
However, uuids are harder to blindly guess and will be useful if later you decide to spread your data out to multiple systems.

```zsh
mix phx.new --binary-id --database postgres grade_tracker 
```


If you are used to rails or django, you'll have some ideas to unlearn.
ORM stands for Object relational mapping.
Elixir on the other hand doesn't have objects in the sense that ruby or python has.
There are no classes or instances of those classes.
You won't find such a thing as an "instance" keyword.
Instead, Elixir gives us [structs](https://elixircasts.io/intro-to-structs).

A struct is a datastructure similar to [Map](https://hexdocs.pm/elixir/1.0.5/Map.html) with a difference that the keys are defined up front.
Ecto gives us a schema which is a struct that maps to a database record.
Unlike an active record model, an ecto schema record has no internal state.
Its simply a data structure that you pass around.

First things first, we need a teacher so lets fire up the generator

```zsh
$mix phx.gen.schema Teacher teachers name:string
* creating lib/grade_tracker/teacher.ex
* creating priv/repo/migrations/20210414205427_create_teachers.exs
```

This generates the following files


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

The schema that was generated is the interface for how we will interact with this table

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
    |> cast(attrs, [:name])
    |> validate_required([:name])
  end
end
```


One refreshing thing in Elixir is that there is an explicit avoidence of magic behavior while managing to avoid excessive boiler plate found in other magic(ie: abstraction) averse languages *cough* **go** *cough*.


Since we are dealing with classes,  it makes sense to add a class table to which will have one or more students.













