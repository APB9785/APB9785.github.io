<h2>Autoscroll for Chatbox or Log Stream</h2>

A common thing for me to implement is a box for chat messages or some kind of logs, where entries keep coming in, getting appended at the bottom, and we want to keep the focus at the bottom of the list, so we can always see the most recent entry.

We know that the list of entries can grow indefinitely, so this is a good use case for [temporary assigns]:
```elixir
def mount(_, _, socket) do
  socket = assign(socket, :logs, Logs.list_all())
  {:ok, socket, temporary_assigns: [logs: []]}
end

def handle_event("new_log_entry", params, socket) do
  new_log = Logs.new!(params)
  {:noreply, assign(socket, logs: [new_log])}
end
```
With our `logs` assign ready to render new entries without storing them server-side, let's make the box to display them, in descending order:
```heex
<div id="log-display" phx-update="append">
  <%= for entry <- @logs do %>
    <div id={"log-#{entry.id}"}>
      <%= entry.message %>
    </div>
  <% end %>
</div>
```
This will work OK until the list of entries grows longer than the box can fit - then we should make it scrollable:  
```heex
<div id="log-display" phx-update="append"
     class="overflow-y-auto">
```
NOTE: I'm using tailwind for CSS, which generates this class as a shortcut for `overflow-y: auto;`


Now the box will not overflow, but the user will have to continually scroll down to see new entries.  We would prefer if the focus was automatically adjusted to show new entries without any user input.  To do this, we can use the [scrollTop](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTop) and [scrollHeight](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollHeight) properties of our window element.  But before we get into that, let's start by creating a JS hook in our `app.js`:
```javascript
let Hooks = {};

Hooks.LogScroll = {
  ...
}

let csrfToken = document.querySelector("meta[name='csrf-token']").getAttribute("content")
let liveSocket = new LiveSocket("/live", Socket, {
  hooks: Hooks,
  params: {_csrf_token: csrfToken}
})
```
Here we've created a Hooks object, which gets passed into the liveSocket.  So far, it contains a single hook called LogScroll, which we must associate with our log box in the template:
```heex
<div id="log-display" phx-update="append" phx-hook="LogScroll"
     class="overflow-y-auto">
```
Now we can get to the logic of the hook.  We want to change the current state of the scroll bar (`scrollTop`) - but this value is measured in pixels, going from top to bottom.  Fortunately, the `scrollHeight` property tells us exactly the height, in pixels, for our full list of entries.  So our goal is accomplished simply:
```javascript
this.el.scrollTop = this.el.scrollHeight;
```
To add this logic to our hook, we can use the `mounted()` callback to run the code as soon as the log box is added to the DOM.  This will ensure that if the page loads with many entries, we will start at the bottom of the list.  But we also want it to run each time a new entry is added, and for that we will use the `updated()` callback.  With both callbacks, our final hook will look like this:
```javascript
Hooks.LogScroll = {
  mounted() {
    this.el.scrollTop = this.el.scrollHeight;
  },
  updated() {
    this.el.scrollTop = this.el.scrollHeight;
  }
}
```

Coming soon: how to disable the AutoScroll effect when the user is scrolling up through the old logs.
