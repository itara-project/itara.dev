# itara.dev
 
Documentation site for [Itara](https://github.com/itara-project/itara) —
topology as a dedicated layer.
 
Built with [Hugo](https://gohugo.io/) and the [Docsy][] theme, hosted on
Netlify.
 
## Running the website locally
 
Install the prerequisites: Node.js and npm (minimum versions enforced at
install time — see `package.json` `engines`), plus Go and Git. Hugo itself
comes from the pinned [hugo-extended][] npm package. On Windows, npm scripts
run under Bash (which ships with [Git for Windows](https://gitforwindows.org/)):
make sure `bash` is on your `PATH`.
 
From the repo root folder, install the site's dependencies:
 
```bash
npm run install:safe
```
 
This performs a clean, script-free install of the pinned dependencies; the
Hugo binary self-installs at first use. If npm blocks a needed install
script, approve it explicitly:
 
```bash
npm run approve:hugo
```
 
Then run:
 
```bash
npm run serve
```
 
Run Hugo through the npm scripts, as above: they put the [Dart Sass][] `sass`
CLI from `node_modules` on the `PATH`; a direct `hugo` invocation fails
without it.
 
## Contributing
 
See [CONTRIBUTING.md](CONTRIBUTING.md).
 
[Docsy]: https://github.com/google/docsy
[Dart Sass]: https://www.docsy.dev/docs/get-started/docsy-as-module/installation-prerequisites/#install-dart-sass
[hugo-extended]: https://www.npmjs.com/package/hugo-extended

<!-- cSpell:ignore hugo docsy -->
