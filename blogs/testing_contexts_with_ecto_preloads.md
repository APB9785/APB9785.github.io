If you add preload functionality to your context, you will also find that some of your tests will be broken, giving an error like:
```
Assertion with == failed
     code:  assert Announcements.list_announcements() == [announcement]
     left:  [
              %MyApp.Announcements.Announcement{
                __meta__: #Ecto.Schema.Metadata<:loaded, "announcements">,
                body: "some body",
                id: 2,
                inserted_at: ~N[2021-06-29 21:24:58],
                title: "some title",
                updated_at: ~N[2021-06-29 21:24:58],
                user: #MyApp.Accounts.User< ... >,
                user_id: 5491
              }
            ]
     right: [
              %MyApp.Announcements.Announcement{
                __meta__: #Ecto.Schema.Metadata<:loaded, "announcements">,
                body: "some body",
                id: 2,
                inserted_at: ~N[2021-06-29 21:24:58],
                title: "some title",
                updated_at: ~N[2021-06-29 21:24:58],
                user: #Ecto.Association.NotLoaded<association :user is not loaded>,
                user_id: 5491
              }
            ]
```
In this simple case, where we already have the User struct from the test setup, we can manually add the data without another DB query:
```
test "list_announcements/0 returns all announcements", %{user: user} do
  announcement = announcement_fixture(user) |> Map.put(:user, user)
  assert Announcements.list_announcements() == [announcement]
end
```
But what if we didn't already have the user data?  We can still ask Ecto to "pre" load it into the Announcement:
```
test "list_announcements/0 returns all announcements" do
  announcement = announcement_fixture(user) |> MyApp.Repo.preload(:user)
  assert Announcements.list_announcements() == [announcement]
end
```
Either way, your test should now pass!
