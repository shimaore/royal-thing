    seem = require 'seem'

    module.exports = seem (config) ->
      debug 'Start', config
      unless config?
        config_file = process.env.CONFIG
        unless config_file?
          throw new Error 'Missing CONFIG environment variable'
        config = JSON.parse yield fs.readFileAsync config_file, encoding:'utf8'

      design_doc = "_design/#{pkg.name}"
      db = new PouchDB config.provisioning_admin

      save = seem (seq) ->
        debug 'save', {seq}
        config.update_seq = seq
        return false if process.env.CONFIG_READONLY is 'true'
        return false unless config_file?
        yield fs
          .writeFileAsync config_file, JSON.stringify(config, null, '  '), encoding:'utf8'
          .catch (error) ->
            debug "save: #{config_file}: #{error.stack ? error}"
            null

On first run (no `update_seq` in configuration) retrieve the last `update_seq` and save it in the configuration file.

      if not config.update_seq?
        debug 'install: save current update_seq'
        info = yield db.info()
        yield save info.update_seq

Since we need to monitor only changes in global numbers, add a filter in the provisioning database towards that goal.

      doc = yield db
        .get design_doc
        .catch ->
          _id:design_doc
      doc.filters =
        global_numbers: fun ({_id}) ->
          _id.match /^number:\d+$/
      yield db
        .put doc
        .catch (error) ->
          debug "put failed: #{error.stack ? error}", doc

      db.close()
      {config,save}

    path = require 'path'
    Promise = require 'bluebird'
    fs = Promise.promisifyAll require 'fs'
    pkg = require './package.json'
    PouchDB = require 'pouchdb'
    debug = (require 'debug') "#{pkg.name}:install"

    fun = (f) -> "(#{f})"
