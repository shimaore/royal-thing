Since we need to monitor only changes in global numbers, add a filter in the provisioning database towards that goal.

    module.exports = ->
      config = require './config.json'

      design_doc = "_design/#{pkg.name}"
      db = new PouchDB config.provisioning_admin

On first run (no `update_seq` in configuration) retrieve the last `update_seq` and save it in the configuration file.

      save = (seq) ->
        config.update_seq = seq
        fs.writeFileAsync (path.join (path.dirname module.filename), './config.json'), JSON.stringify config, null, '  '

      db.info()
      .then (info) ->
        if not config.update_seq?
          save info.update_seq
      .then ->
        db.get design_doc
      .catch ->
        _id:design_doc
      .then (doc) ->
        doc.filters =
          global_numbers: fun ({_id}) ->
            _id.match /^number:\d+$/
        db.put doc
      .then ->
        {config,save}

    path = require 'path'
    Promise = require 'bluebird'
    fs = Promise.promisifyAll require 'fs'
    pkg = require './package.json'
    PouchDB = require 'pouchdb'

    fun = (f) -> "(#{f})"
