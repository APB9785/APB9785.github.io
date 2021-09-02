# Deploying to Heroku with Phoenix 1.6 and TailwindCSS

## Upgrading to 1.6

If your app is not using Phoenix 1.6 yet, follow the official upgrade guide [here](https://gist.github.com/chrismccord/2ab350f154235ad4a4d0f4de6decba7b).  If you have errors when running `mix assets.deploy`, [these](https://github.com/APB9785/APB9785.github.io/blob/master/blogs/upgrade_phoenix_1_6_esbuild_errors.md) extra steps may be helpful.

## Installing Tailwind

[Sergio Tapia](https://github.com/sergiotapia) has already made an [excellent guide](https://sergiotapia.com/phoenix-160-liveview-esbuild-tailwind-jit-alpinejs-a-brief-tutorial) to installing Tailwind, and there is even a generator [phx.gen.tailwind](https://github.com/kevinlang/phx_gen_tailwind) by [Kevin Lang](https://github.com/kevinlang) to automatically install Tailwind.  So I will not go into detail about this, but move straight on to the focus of this guide, deployment.

## Deploying to Heroku

The Phoenix team has updated the [official deployment guide](https://hexdocs.pm/phoenix/1.6.0-rc.0/heroku.html#create-and-set-up-heroku-application) for 1.6, so let's use that as our starting point.  After the Heroku setup, adding the Elixir and Phoenix buildpacks + config files, updating the project configs, and setting environment variables, you should be ready to deploy.  But after running the `git push`, you'll likely receive an error saying
```
-----> Executing hook after compile: mix assets.deploy && rm -f _build/esbuild
sh: 1: npm: not found
```
This is caused by the line added to `elixir_buildpack.config`:
```
hook_post_compile="mix assets.deploy && rm -f _build/esbuild"
```
This task `assets.deploy` is necessary, but when we installed Tailwind, this task changed to include `"cmd --cd assets npm run deploy"`, which requires node package manager (npm).  Since node (and npm) are installed by the Phoenix buildpack, and the Phoenix buildpack runs AFTER the Elixir buildpack (if you followed the deployment guide properly), then clearly npm will not be available when the Elixir buildpack attempts to run the `hook_post_compile`.

To solve this, we simply remove/comment out the aforementioned line from the Elixir buildpack.  Now the deployment should run successfully.  If you still receive compilation errors, here are some things to try:
* Check the `node_version` in `phoenix_static_buildpack.config`.  I personally recommend using 15.7.0 for Phoenix projects.
* Add `clean_cache=true` to `phoenix_static_buildpack.config`
* Delete `mix.lock` and `assets/package-lock.json`, then run `mix deps.get` and `npm install --prefix assets` to re-create them.

Now hopefully you have a successful deployment, but there is still one problem:  If you open the deployed app in your browser, and look at the javascript console, you'll notice that your assets are not found (404).  This is because we removed `mix assets.deploy` (which runs `esbuild`) from the Elixir buildpack.  So the last thing we must do is add this task to the **Phoenix** buildpack, where it will run **after** node+npm are installed.  First let's take a look at the default compile script used by the buildpack:
```
if [ -f "$assets_dir/yarn.lock" ]; then
  yarn deploy
else
  npm run deploy
fi

cd $phoenix_dir

mix "${phoenix_ex}.digest"

if mix help "${phoenix_ex}.digest.clean" 1>/dev/null 2>&1; then
  mix "${phoenix_ex}.digest.clean"
fi
```
Notice the first task is to run `npm run deploy` (or `yarn deploy` if you are using yarn).  But this is already handled in `mix assets.deploy`, so we can ignore the whole if/else statement, and instead add `mix assets.deploy` and `rm -f _build/esbuild` (the two commands we removed from the Elixir buildpack):
```
cd $phoenix_dir

mix assets.deploy
rm -f _build/esbuild

mix "${phoenix_ex}.digest"

if mix help "${phoenix_ex}.digest.clean" 1>/dev/null 2>&1; then
  mix "${phoenix_ex}.digest.clean"
fi
```
Save this as `compile` in the root directory of your project, commit the change, and push it to Heroku.  If everything went well, your app should be running smoothly with Phoenix 1.6, esbuild, and TailwindCSS.  If not, consider making a post on the [Elixir forum](https://elixirforum.com), where there are many talented and experienced community members who can help you troubleshoot.
