<h2>Testing Contexts with phx.gen.auth</h2>

One issue many beginners run into when adding phx.gen.auth to their application, is that the test suite automatically provided by the context generator is insufficient to handle a Context which references a user account (such as a `Comment` which belongs to an `Author` with `author_id`).  Running the tests could result in various errors such as:   

```
** (MatchError) no match of right hand side value:
{:error, #Ecto.Changeset<..., errors: [author_id: {"can't be blank", [validation: :required]}], ...}
```
or

```
** (Ecto.ConstraintError) constraint error when attempting to insert struct:
    * comments_author_id_fkey (foreign_key_constraint)
```

The root of the problem lies with how the test data is created.  In the test file for each context, you should have something like:

```
@valid_attrs %{title: "some title", body: "some body", author_id: 42}
@update_attrs %{title: "some updated title", body: "some updated body", author_id: 43}
@invalid_attrs %{title: nil, body: nil, author_id: nil}

def comment_fixture(attrs \\ %{}) do
  {:ok, comment} =
    attrs
    |> Enum.into(@valid_attrs)
    |> Comments.create_comment()

  comment
end
```
Or maybe the `author_id` field is missing altogether.

Either way, we need to update our tests to first create a user account, and then when `comment_fixture` is called, it should use the id of the user when creating the Comment.

To create a user account, there should be a function in `test/support/fixtures/accounts_fixtures.ex` called `user_fixture/1`.

> NOTE: If you've added any extra required fields to the User schema, you will have to add those to the valid_user_attributes, so the fixture can successfully create valid user accounts.

Assuming the `user_fixture` looks okay, let's use it in `comments_test.exs`:
```
describe "comments" do
  import MyApp.AccountsFixtures, only: [user_fixture: 0]

  alias MyApp.Comments.Comment
  
  @valid_attrs %{title: "some title", body: "some body"}
  @update_attrs %{title: "some updated title", body: "some updated body"}
  @invalid_attrs %{title: nil, body: nil, author_id: nil}

  setup do
    %{user: user_fixture()}
  end

  def comment_fixture(user) do
    {:ok, comment} =
      %{author_id: user.id}
      |> Enum.into(@valid_attrs)
      |> Comments.create_comment()

    comment
  end
```
First, we import the function with arity 0 (because we will not be passing any input params).  Then, since all of our Comment tests will require a user to be created, we can use `setup` to call `user_fixture` automatically, without needing to add an additional line of code to the tests.  And finally, we change the `comment_fixture` to require a user account as the input parameter.

To implement these changes into the tests, all we need to do is assign the user to a variable in the test declaration, and then pass it into `comment_fixture/1`:
```
test "list_comments/0 returns all comments", %{user: user} do
  comment = comment_fixture(user)
  assert Comments.list_comments() == [comment]
end
```

Now most of our tests should pass, but there are probably still a few failures...

You'll notice that some of our tests call `Comments.create_comment/1` directly, using the module attributes.  This will not be adequate because there is no way for the module to know at compile time what the author's id will be.  Instead, let's follow the example from `comment_fixture/1` and use `Enum.into/2` to create the full set of attrs.  Then, that will get passed to `Comments.create_comment/1` instead:
```
test "create_comment/1 with valid data creates a comment", %{user: user} do
  attrs = Enum.into(%{author_id: user.id}, @valid_attrs)
  assert {:ok, %Comment{} = comment} = Comments.create_comment(attrs)
  assert comment.title == "some title"
  assert comment.body == "some body"
  assert comment.author_id == user.id
end
```
After ensuring each test has been updated accordingly, we should be back to 100% passing tests for this Context!
