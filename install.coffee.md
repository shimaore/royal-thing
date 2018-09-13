    module.exports = (host,db) ->
      debug 'Start', host

      local_uri = new URL "_local/royal-#{host}", db.uri+'/'

      save = (seq) ->
        debug 'save', {seq}
        doc = await db.agent
          .get local_uri
          .accept 'json'
          .then ({body}) -> body
          .catch -> {}

        if not seq?
          return doc

        doc.update_seq = seq
        await db.agent
          .put local_uri
          .send doc

On first run, retrieve the last `update_seq` and save it.

      {update_seq} = await save()

      if not update_seq?
        debug 'install: save current update_seq'
        info = await db.info()
        await save info.update_seq
      else
        debug 'Re-using', {update_seq}

      {save}

    {URL} = require 'url'
    pkg = require './package.json'
    debug = (require 'debug') "#{pkg.name}:install:trace"
