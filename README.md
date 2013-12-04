<img src="img/haskell.png" alt="Styleguide Rails Logo" align="left" />
<img src="img/heroku.png" alt="Styleguide Rails Logo" align="left" />

<h2 align="right">Deploy Haskell apps to Heroku</h2>

<br />

This buildpack supports frameworks like Yesod, Snap, and Happstack with
the latest stable GHC binaries. Putting Haskell web applications online
should be easy, and now it is. Try it for yourself.

### Example: deploying a Snap app

Here's how to go from zero to "hello world" on Heroku. You'll need to
install the [Haskell Platform](http://www.haskell.org/platform/) and
the [Heroku Toolbelt](https://toolbelt.herokuapp.com/) on your local
machine, then do this:

```sh
# Generate a barebones snap app called snapdemo

mkdir snapdemo && cd $_
cabal sandbox init
cabal install snap
snap init barebones

# Tell Heroku how to start the server

echo 'web: cabal run -- -p $PORT' > Procfile

# Create a git repo and deploy!

git init .
git add *
git commit -m 'Initial commit'

heroku create --stack=cedar --buildpack https://github.com/begriffs/heroku-buildpack-ghc.git
git push heroku master
```

**The first deploy is very slow** as the environment downloads and
bootstraps. Subsequent deploys use cached binaries and go much faster.

### Interacting with a running app

```sh
heroku run bash         # shell access
heroku run cabal repl   # Haskell repl with all app modules loaded
```

### Benefits of this buildpack

* Latest binaries: GHC 7.6.3, Cabal 1.18.0.2
* Uses cabal >=1.18 features to run the app and repl
* Exposes Haskell platform binaries to your app and scripts
* Bootstraps from the standard GHC distribution

### Contributing

There are a number of ways to improve this buildpack. Please see the
Github issues for ideas.

In order to contribute to the build script it will help to understand
how Heroku's deployment process works, and how that affects GHC. Heroku
provides three areas for storing files during build: a cache directory,
a working directory, and a build directory.

The cache, called `$CACHE_DIR` in the script, persists between
deployments. We use it to avoid building binaries more than once. The
working directory, called `$WORKING_HOME`, does not persist between
builds, or even after the build script is done. Seems we should avoid
this area, right? Well GHC has some idiosyncracies that make this area
quite useful as you will see. Finally the build destination directory,
`$BUILD_DIR`, holds the git repo the user pushes and gets copied into
what will be `/app` in the deployed application. During build it lives
in a weird nonce filename.

We want to use GHC binaries at build time to compile the app, and we
would also like those binaries to be available in the app environment
after deployment (so people can use `runhaskell` or `cabal repl`). Seems
like we should install directly to `$BUILD_DIR`. There's one problem:
GHC breaks if you move it to a new path after installation because many
of its binaries are just scripts with hard-coded full paths in them to
other GHC files. And as you remember, Heroku is going to move things in
`$BUILD_DIR` to `/app`. So the trick will be to install to `/app` in the
working directory, use GHC there, then copy that installation to the
build directory which will be renamed in the deployed application and
not notice it has been moved.

GHC is also sensitive to having `libgmp` named just right. We don't
have privileges to adjust `/usr/lib` in the deployed app so we create a
symbolic link in a place we are permitted and set linker variables in
the shell so that everything can build.

### Thanks

Thanks to Brian McKenna and others for their work on
[heroku-buildpack-haskell](https://github.com/puffnfresh/heroku-buildpack-haskell)
which inspired and informed this buildpack. Their project got too forked
and fragmented over time, so I thought it best to start fresh with a new
name and code. For a history of that project's contributions see [this article]
(http://blog.begriffs.com/2013/08/haskell-on-heroku-omg-lets-get-this.html).
