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

**The first deploy is slowest** as the environment downloads and
bootstraps. Subsequent deploys use cached binaries and cached cabal
packages to go faster.

### Beating the Fifteen-Minute Build Limit

The first time you try to deploy a big framework like Yesod the
compilation can take so long that Heroku cuts it off. If this happens
fear not, you can build your app with an Anvil server.

```sh
heroku plugins:install https://github.com/ddollar/heroku-anvil
heroku build -r -b https://github.com/begriffs/heroku-buildpack-ghc.git
```

After the first deploy using Anvil you can go back to the regular deploy
process. This is because the cabal sandbox is cached between builds so
future builds are incremental and fast.

### Locking Package Versions

Cabal sometimes gets confused on Heroku and tries installing outdated
packages. If you have your app working locally you can constrain the
remote package versions to match your local environment. Just do this:

```sh
cabal install cabal-constraints
cabal-constraints dist/dist-sandbox-*/setup-config > cabal.config
git add cabal.config

# commit and push to fix remote build
```

Using cabal-constraints requires the Cabal version be exactly 1.18.1.2.
This process will be improved in the Cabal 1.20 release, once
cabal-constraints is deprecated in favor of the upcoming cabal-install
freeze command. For more info, see the docs for
[cabal-constraints](https://github.com/benarmston/cabal-constraints)

### Clearing the build cache

If you want to force everything to reinstall from scratch, set a Heroku
environment variable prior to pushing the deploy.

```sh
heroku config:set CLEAR_BUILDPACK_CACHE=1

# allow the buildpack to see environment vars
heroku labs:enable user-env-compile
```

### Interacting with a running app

```sh
heroku run bash         # shell access
heroku run cabal repl   # Haskell repl with all app modules loaded
```

### Benefits of this buildpack

* **Latest binaries: GHC 7.6.3, Cabal 1.18.0.2**
* Uses cabal >=1.18 features to run the app and repl
* Exposes Haskell platform binaries to your app and scripts
* Uses prebuilt binaries for speed but...
* ...can fall back to building the standard GHC distribution

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

### Building new binaries for Heroku

As new versions of GHC and Cabal are released we should build them for
Heroku and put them on S3 to speed up future deploys for everyone. Luckily
the buildpack can do the building too.

Adjust the `GHC_VERSION` and `CABAL_VERSION` in `bin/compile` and
`bin/release` then deploy. It will build the new binaries from the
standard GHC distribution. Then copy the results to S3 like this:

```sh
heroku run bash
# now SSH'd into the server

cd /app/vendor

curl -L http://softlayer-ams.dl.sourceforge.net/project/s3tools/s3cmd/1.5.0-alpha1/s3cmd-1.5.0-alpha1.tar.gz | tar zx
s3cmd-1.5.0-alpha1/s3cmd --configure
# ^^^ answer the configuration questions

tar zcf heroku-ghc-[VERSION].tar.gz ghc-[VERSION]/
tar zcf heroku-cabal-install-[VERSION].tar.gz cabal-install-[VERSION]/

s3cmd-1.5.0-alpha1/s3cmd put heroku-ghc-[VERSION].tar.gz s3://[BUCKET]
s3cmd-1.5.0-alpha1/s3cmd put heroku-cabal-install-[VERSION].tar.gz s3://[BUCKET]
```

### Addendum: Character encoding on Heroku

Haskell uses the default system character encoding unless you specify otherwise.
You probably want this to be UTF-8, so to avoid problems set the following
option:

    heroku config:add LANG=en_US.UTF-8

### Thanks

Thanks to Brian McKenna and others for their work on
[heroku-buildpack-haskell](https://github.com/puffnfresh/heroku-buildpack-haskell)
which inspired and informed this buildpack. For a history of that project's
contributions and ideas see [this article]
(http://blog.begriffs.com/2013/08/haskell-on-heroku-omg-lets-get-this.html).
