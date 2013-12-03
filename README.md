<img src="img/haskell.png" alt="Styleguide Rails Logo" align="left" />
<img src="img/heroku.png" alt="Styleguide Rails Logo" align="left" />

## Deploy Haskell apps to Heroku

<br />

Publish Haskell apps to the web with a few commands. This buildpack
provides GHC 7.6.3 and Cabal 1.18 for your app, and maintains up a Cabal
sandbox for app dependencies.

## How to deploy a Snap app

Here's how to go from zero to "hello world" on Heroku.

```sh
# Generate a barebones snap app called snapdemo

mkdir snapdemo && cd $_
cabal sandbox init
cabal install snap
snap init barebones

# Tell Heroku how to start the server

echo 'web: cabal run snapdemo -- -p $PORT' > Procfile

# Set up your repo and deploy!

git init .
git add .
git commit -m 'Initial commit'

heroku create --stack=cedar --buildpack https://github.com/begriffs/heroku-buildpack-ghc.git
git push heroku master
```

## Benefits of this buildpack

* 

## Contributing

## Process of 
