# Heroku buildpack: Clojure

This is a Heroku buildpack for Clojure apps. It uses
[Leiningen](https://github.com/technomancy/leiningen).

Note that you don't have to do anything special to use this buildpack
with Clojure apps on Heroku; it will be used by default for all
projects containing a project.clj file, though it may be an older
revision than current master. This repository is made available so
users can fork for their own needs and contribute patches back.

## Usage

Example usage for an app already stored in git:

    $ tree
    |-- Procfile
    |-- project.clj
    |-- README
    `-- src
        `-- sample
            `-- core.clj

    $ heroku create --stack cedar --buildpack http://github.com/heroku/heroku-buildpack-clojure.git

    $ git push heroku master
    ...
    -----> Heroku receiving push
    -----> Fetching custom buildpack
    -----> Clojure app detected
    -----> Installing Leiningen
           Downloading: leiningen-1.6.2-standalone.jar
           Downloading: rlwrap-0.3.7
           Writing: lein script
    -----> Installing dependencies with Leiningen
           Running: LEIN_NO_DEV=y lein deps
           Downloading: org/clojure/clojure/1.2.1/clojure-1.2.1.pom from central
           Downloading: org/clojure/clojure/1.2.1/clojure-1.2.1.jar from central
           Copying 1 file to /tmp/build_2e5yol0778bcw/lib
    -----> Discovering process types
           Procfile declares types -> core
    -----> Compiled slug size is 10.0MB
    -----> Launching... done, v4
           http://gentle-water-8841.herokuapp.com deployed to Heroku

The buildpack will detect your app as Clojure if it has a
`project.clj` file in the root. If you use the
[clojure-maven-plugin](https://github.com/talios/clojure-maven-plugin),
[the standard Java buildpack](http://github.com/heroku/heroku-buildpack-java)
should work instead.

This buildpack will call `LEIN_NO_DEV=y lein deps` to copy your
dependencies into the `lib` directory.

## Private Repositories

If you have dependencies which cannot be published to any public Maven
repository, you can use the
[s3-wagon-private](https://github.com/technomancy/s3-wagon-private)
plugin to publish and consume from private repositories hosted on S3.

Currently this requires installing `s3-wagon-private` as a user-level
plugin locally and using the `s3-wagon` branch of the Clojure buildpack:

    $ lein plugin install s3-wagon-private 1.0.0
    $ heroku config:add BUILDPACK_URL=git@github.com:heroku/heroku-buildpack-clojure.git#s3-wagon

Future versions of Leiningen will allow you to declare
`s3-wagon-private` in `project.clj`, but for the time being you may
want to include this warning so you will get more helpful error
messages when it's missing:

```clj
(try (resolve 's3.wagon.private.PrivateWagon)
     (catch java.lang.ClassNotFoundException _
       (println "WARNING: You appear to be missing s3-private-wagon.")
       (println "To install it: lein plugin install s3-private-wagon 1.0.0")))
```

It's [recommended](http://www.12factor.net/config) to keep your
repository credentials out of source control. You'll need to
[enable user_env_compile](http://devcenter.heroku.com/articles/labs-user-env-compile)
for your app to expose config variables at compile time:

    $ heroku plugins:install http://github.com/heroku/heroku-labs.git # if needed
    $ heroku labs:enable user_env_compile

Then you can add the `AWS_ACCESS_KEY` and `AWS_SECRET_KEY` config values:

    $ heroku config:add AWS_ACCESS_KEY=[...] AWS_SECRET_KEY=[...]

Finally you'll need to read these values into Leiningen. You can
either embed calls to `System/getenv` into project.clj directly or
check in a `.lein-heroku-init.clj` file that will be copied to
`~/.lein/init.clj` in the slug compilation environment.

```clj
(def leiningen-auth {"s3p://secret-bucket/releases"
                     {:username (System/getenv "AWS_ACCESS_KEY")
                      :passphrase (System/getenv "AWS_SECRET_KEY")}})
```

## Hacking

To change this buildpack, fork it on GitHub. Push up changes to your
fork, then create a test app with `--buildpack YOUR_GITHUB_URL` and
push to it. If you already have an existing app you may use
`heroku config:add BUILDPACK_URL=YOUR_GITHUB_URL` instead.

For example, you could adapt it to generate an uberjar at build time.

Open `bin/compile` in your editor, and replace the block labeled
"fetch deps with lein" with something like this:

    echo "-----> Generating uberjar with Leiningen:"
    echo "       Running: LEIN_NO_DEV=y lein uberjar"
    cd $BUILD_DIR
    PATH=.lein/bin:/usr/local/bin:/usr/bin:/bin JAVA_OPTS="-Xmx500m -Duser.home=$BUILD_DIR" LEIN_NO_DEV=y lein uberjar 2>&1 | sed -u 's/^/       /'
    if [ "${PIPESTATUS[*]}" != "0 0" ]; then
      echo " !     Failed to create uberjar with Leiningen"
      exit 1
    fi

The `LEIN_NO_DEV` environment variable will cause Leiningen to keep
the test directories and dev dependencies off the classpath, so be
sure to set it for every `lein` invocation.

Commit and push the changes to your buildpack to your GitHub fork,
then push your sample app to Heroku to test. You should see:

    -----> Generating uberjar with Leiningen:

## License

Copyright © 2011 Heroku, Inc.

Distributed under the MIT/X11 license:

Permission is hereby granted, free of charge, to any person
obtaining a copy of this software and associated documentation
files (the "Software"), to deal in the Software without
restriction, including without limitation the rights to use,
copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following
conditions:
 
The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.
 
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES
OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
OTHER DEALINGS IN THE SOFTWARE.
