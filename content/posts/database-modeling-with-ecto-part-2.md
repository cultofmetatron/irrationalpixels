---
title: "Database Modeling With Ecto Part 2 - Relationships"
date: 2021-05-09T21:06:14-04:00
tag: elixir
draft: true
---

## contents
* creating out first relationship
* has_many through
* aggregate functions


Previously, we created the inital pheonix app and created a `Teacher` struct and table.
We went over how to perform the basic CRUD operations as well as how to do some basic tests to verify the functionality.
In part 2, we're going to be adding more structs and defining relationships between them.
We'll also go over how to fetch records with their associations.

If you haven't been following along, you can checkout the code from github and fetch the branch.

```bash
git clone git@github.com:cultofmetatron/database-modeling-with-ecto-example.git
git checkout part1
```

## has many and belongs to

We now have a `Teacher` ecto schema as an interface to out `teachers` table.
Lets implement a class.
A `Class` at most belongs to one `Teacher`.
The standard way to setup a relationship is through foreign keys.
That is a column that matches the a column on another table which msut be unique on the other table.
The type of the foreign key column must match the type of the key on the foreign table that its referencing.

We will create a column `teacher_id` on the `classes` table will have a *foreign key constraint* on the `id` columnn of the `teachers` table.
Since the `id` column of `teachers` is a uuid, the column here will also be a `uuid`.
Being the primary key of the table, it alrady satisfies the aformentioned uniqueness contraint.
In ecto, we call it a `:binary_id`.

```zsh
$ mix phx.gen.schema Class classes name:string teacher_id:references:teachers subject:string active:boolean
* creating lib/grade_tracker/class.ex
* creating priv/repo/migrations/20210515195753_create_classes.exs

Remember to update your repository by running migrations:

    $ mix ecto.migrate

```

Now that we've created the associated class and migrations, we need to make some small adjustments.

First lets look at the migration

```elxiir
defmodule GradeTracker.Repo.Migrations.CreateClasses do
  use Ecto.Migration

  def change do
    create table(:classes, primary_key: false) do
      add :id, :binary_id, primary_key: true
      add :name, :string
      add :subject, :string
      add :active, :boolean, default: false, null: false
      add :teacher_id, references(:teachers, on_delete: :nothing, type: :binary_id)

      timestamps()
    end

    create index(:classes, [:teacher_id])
  end
end

```

Same as teacher, we have an id set with `:binary_id` which coresponds to the `UUID` type in postgres.
note, that part for teacher id, we have an `on_delete: :nothing`.
Change that to

```elixir

add :teacher_id, references(:teachers, on_delete: :nilify_all, type: :binary_id)

```

The call to [*references/2*](https://hexdocs.pm/ecto_sql/Ecto.Migration.html#references/2) creates the foreign key constraint.
What this means is any non null values for the teacher_id column of this table **MUST** corespond to a a record in the `teachers` table in the referenced column.
This leads to some edge cases that must be accounted for.
What happens if the referenced teacher record is deleted?
All classes referring to that eacher would have value for `teacher_id` that violates the contraint.
If you have a a few classes that belong to this teacher and you try to remove the teacher, you will get an error from the database.
This is rarely what I want.

The `on_delete` specifies the action to take place if the referenced record is deleted.
If we leave the value as `:nothing`, we will get the error I mentioned.
In this case, I am switching it to `:nilify_all`.
This makes the class without a teacher.
In a real world scneario, I could make a dashbaord showing the orphned classes prompting them to assign a teacher.
Alternativly, I could set `delete_all` which would remove the classes that reference the teacher.
What you pick for your relationships depend entirely on your use case.


By default, the key is set to `id`. you can overide that by passing in `:column`.
We can also add it to make the associated column more explicit.

If We wanted to make it so that all classes MUST have a teacher id passed in, you could pass `null: true` to the column creation.
If you do that, `nilify_all` will not work as it will violate the null column contraint.
You will have to either `delete_all` or have your app force the user to reasign the class before removing a teacher.

```elixir

# does the same thing as above
add :teacher_id, references(:teachers, on_delete: :nilify_all, type: :binary_id, column: :id)

```

One more thing, we should make that default for the `active` column to be true.

your migration should look like this:
```elixir

defmodule GradeTracker.Repo.Migrations.CreateClasses do
  use Ecto.Migration

  def change do
    create table(:classes, primary_key: false) do
      add :id, :binary_id, primary_key: true
      add :name, :string
      add :subject, :string
      add :active, :boolean, default: true, null: false
      add :teacher_id, references(:teachers, on_delete: :delete_all, type: :binary_id)

      timestamps()
    end

    create index(:classes, [:teacher_id])
  end
end


```


Now lets take a look at the schema.

```elixir

defmodule GradeTracker.Class do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  schema "classes" do
    field :active, :boolean, default: false
    field :name, :string
    field :subject, :string
    field :teacher_id, :binary_id

    timestamps()
  end

  @doc false
  def changeset(class, attrs) do
    class
    |> cast(attrs, [:name, :subject, :active])
    |> validate_required([:name, :subject, :active])
  end
end

```

By default, the schema specifies no relationship to Teacher.
We need to modify this schema by replaceing the `teacher_id` field with [*belongs_to/3*](https://hexdocs.pm/ecto/Ecto.Schema.html#belongs_to/3).
it'll probably be a good idea to add a foreing constraint to the validation code in teh changeset as well.
Additionally, we'll remove active as a required field since we have a default and add the `:teacher_id` as a castable attribute.

```elixir

defmodule GradeTracker.Class do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
  schema "classes" do
    field :active, :boolean, default: true
    field :name, :string
    field :subject, :string


    belongs_to :teacher, GradeTracker.Teacher

    timestamps()
  end

  @doc false
  def changeset(class, attrs) do
    class
    |> cast(attrs, [:name, :subject, :active, :teacher_id])
    |> validate_required([:name, :subject])
    |> foreign_key_constraint(:teacher_id)
  end
end


```

I will note that adding `:teacher_id` is optional and may not be appropriate for your use case.
the Belongs to adds other operations we will use to link classes to their associated teachers.
By adding the column directly, you allow the column to be set directly.
This means you should do due diligence if for instance, you take params input from a web request and pass it directly in when creating a changeset. I'll go more detail into that.

Suffice to say, you should watch user inputted data to make sure that the value being inputted is appropiate.

Now we we add a [*has_many/3*](https://hexdocs.pm/ecto/Ecto.Schema.html#has_many/3) to the Teacher.


