{spawn, exec} = require 'child_process'
fs = require 'fs'
log = console.log
remove = require 'remove'

task 'build', ->
  build()

task 'test', ->
  vendor ->
    build ->
      ssl_cert ->
        tokens ->
          run 'mocha --colors --require test/js/helper.js test/js/*test.js'

task 'webtest', ->
  vendor ->
    build ->
      ssl_cert ->
        tokens ->
          webFileServer = require './test/js/web_file_server.js'
          webFileServer.openBrowser()

task 'cert', ->
  remove.removeSync 'test/ssl', ignoreMissing: true
  ssl_cert()

task 'vendor', ->
  remove.removeSync './test/vendor', ignoreMissing: true
  vendor()

task 'tokens', ->
  remove.removeSync './test/.token', ignoreMissing: true
  build ->
    tokens ->
      process.exit 0

task 'extension', ->
  run 'coffee --compile test/chrome_extension/*.coffee'

build = (callback) ->
  # Compile without --join for decent error messages.
  run 'coffee --output tmp --compile src/*.coffee', ->
    run 'coffee --output lib --compile --join dropbox.js src/*.coffee', ->
      # Minify the javascript, for browser distribution.
      run 'uglifyjs --no-copyright -o lib/dropbox.min.js lib/dropbox.js', ->
        run 'coffee --output test/js --compile test/src/*.coffee',
            callback

ssl_cert = (callback) ->
  fs.mkdirSync 'test/ssl' unless fs.existsSync 'test/ssl'
  if fs.existsSync 'test/ssl/cert.pem'
    callback() if callback?
    return

  run 'openssl req -new -x509 -days 365 -nodes -batch ' +
      '-out test/ssl/cert.pem -keyout test/ssl/cert.pem ' +
      '-subj /O=dropbox.js/OU=Testing/CN=localhost ', callback

vendor = (callback) ->
  # All the files will be dumped here.
  fs.mkdirSync 'test/vendor' unless fs.existsSync 'test/vendor'

  # Embed the binary test image into a 7-bit ASCII JavaScript.
  bytes = fs.readFileSync 'test/binary/dropbox.png'
  fragments = []
  for i in [0...bytes.length]
    fragment = bytes.readUInt8(i).toString 16
    while fragment.length < 4
      fragment = '0' + fragment
    fragments.push "\\u#{fragment}"
  js = "window.testImageBytes = \"#{fragments.join('')}\";"
  fs.writeFileSync 'test/vendor/favicon.js', js

  # chai.js ships different builds for browsers vs node.js
  download 'http://chaijs.com/chai.js', 'test/vendor/chai.js', ->
    # sinon.js also ships special builds for browsers, and separate code for IE
    download 'http://sinonjs.org/releases/sinon.js', 'test/vendor/sinon.js', ->
      download 'http://sinonjs.org/releases/sinon-ie.js',
               'test/vendor/sinon-ie.js', callback

tokens = (callback) ->
  TokenStash = require './test/js/token_stash.js'
  tokenStash = new TokenStash
  (new TokenStash()).get ->
    callback() if callback?

run = (args...) ->
  for a in args
    switch typeof a
      when 'string' then command = a
      when 'object'
        if a instanceof Array then params = a
        else options = a
      when 'function' then callback = a

  command += ' ' + params.join ' ' if params?
  cmd = spawn '/bin/sh', ['-c', command], options
  cmd.stdout.on 'data', (data) -> process.stdout.write data
  cmd.stderr.on 'data', (data) -> process.stderr.write data
  process.on 'SIGHUP', -> cmd.kill()
  cmd.on 'exit', (code) -> callback() if callback? and code is 0

download = (url, file, callback) ->
  if fs.existsSync file
    callback() if callback?
    return

  run "curl -o #{file} #{url}", callback
