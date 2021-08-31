# Fixing esbuild errors when upgrading to Phoenix 1.6

## Example

Following along with [Chris McCord's upgrade guide](https://gist.github.com/chrismccord/2ab350f154235ad4a4d0f4de6decba7b), after running `mix assets.deploy` you might receive an error like
```
14:38:05.585 [debug] Downloading esbuild from https://registry.npmjs.org/esbuild-darwin-64/-/esbuild-darwin-64-0.12.18.tgz
 > js/app.js:17:22: error: Could not resolve "nprogress" (mark it as external to exclude it from the bundle)
    17 │ import NProgress from "nprogress"
       ╵                       ~~~~~~~~~~~

 > js/app.js:4:7: error: No loader is configured for ".scss" files: css/app.scss
    4 │ import "../css/app.scss"
      ╵        ~~~~~~~~~~~~~~~~~

2 errors
** (Mix) `mix esbuild default --minify` exited with 1
```
This is the error we will resolve in this post.

## Upgrading app.js

Since our error messages trace back to `js/app.js`, let's start there.  For comparison, I generated a brand-new 1.6 project with https://phx.new/.  If I didn't have any customizations made to `app.js`, I could simply replace my file with the new one.  But in my project I have several JS hooks in that file, so we're only going to copy over the diffs.

Our old file had these imports at the top:
```
import "../css/app.scss"
import "phoenix_html"
import {Socket} from "phoenix"
import NProgress from "nprogress"
import {LiveSocket} from "phoenix_live_view"
```

But in 1.6, `nprogress` is replaced by `topbar`, and `app.scss` is renamed to `app.css`.  Making these changes, our `app.js` imports should now look like:
```
import "../css/app.css"
import "phoenix_html"
import {Socket} from "phoenix"
import {LiveSocket} from "phoenix_live_view"
import topbar from "../vendor/topbar"
```

The other thing we need to do now is replace the `NProgress` configuration with `topbar`.  Currently we have something like:
```
// Show progress bar on live navigation and form submits
window.addEventListener("phx:page-loading-start", info => NProgress.start())
window.addEventListener("phx:page-loading-stop", info => NProgress.done())
```
Which should be replaced by:
```
topbar.config({barColors: {0: "#29d"}, shadowColor: "rgba(0, 0, 0, .3)"})
window.addEventListener("phx:page-loading-start", info => topbar.show())
window.addEventListener("phx:page-loading-stop", info => topbar.hide())
```

## app.css

Once again, if you haven't made any changes to your `app.scss` file, you can simply replace it with a fresh `app.css`.  But if you have, just rename the file to use the proper extension `.css`

## vendor/topbar.js

If we try to run `mix assets.deploy` now, we will get an error like
```
 > js/app.js:27:19: error: Could not resolve "../vendor/topbar"
    27 │ import topbar from "../vendor/topbar"
       ╵                    ~~~~~~~~~~~~~~~~~~
```
This is because the file `topbar.js` is missing from the `assets/vendor` folder.  You can either copy the file from a new 1.6 project, or manually create the file and paste the contents from [here](https://github.com/buunguyen/topbar/blob/master/topbar.js).

## Success

Hopefully you can now run `mix assets.deploy` and see a successful result:
```
mix assets.deploy

  ../priv/static/assets/app.js   79.4kb
  ../priv/static/assets/app.css   1.0kb

⚡ Done in 65ms
Check your digested files at "priv/static"
```
