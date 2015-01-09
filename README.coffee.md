A process that restarts the local registrant when needed
========================================================

    needed = require './needed'
    install = require './install'

    run = (restart) ->
      install()
      .then ({config,save}) ->
        db = new PouchDB config.provisioning

Restarting the process & saving the update sequence
---------------------------------------------------

        restart_needed = false
        new_seq = null

        save_new_seq = ->
          if new_seq?
            save new_seq
            .then ->
              new_seq = null

Only restart at intervals, not on every change.

        interval = setInterval ->
          try
            do on_interval
          catch error
            logger.error "#{pkg.name}: error: #{error}"
        , config.interval ? 61*second

        on_interval = ->
          if restart_needed
            logger.info "#{pkg.name}: calling `restart`."
            restart config, cancel
            .then ->
              restart_needed = false
              save_new_seq()
            .then ->
              logger.info "#{pkg.name}: restart completed."
          else
            save_new_seq()

Especially for tests, we need to provide a way to cancel; `cancel` is given as argument to the `restart` handler, and is returned by `run`.

        cancel = ->
          changes.cancel()
          clearInterval interval

Monitoring changes
------------------

        changes = db.changes
          since: config.update_seq ? 0
          include_docs: true
          live: true
          filter: "#{pkg.name}/global_numbers"

There are various cases where a restart might be needed. The most common ones are defined in the `needed` module. Here we deal with cases were records might not be available, etc.

        .on 'change', ({id,seq,doc}) ->
          new_doc = doc

First if the record was deleted then obviously it was modified and we should restart.

          if not new_doc? or new_doc._deleted
            restart_needed = true
            new_seq = seq
            return

We need to provide `needed` with both the previous record and the current record. Let's try to retrieve the previous record by querying for `revs_info`.

          db.get id,
            revs_info: true

If we can't access the current record for whatever reason, simply re-use the document provided by `change`.

          .catch (error) ->
            logger.error "#{pkg.name}: rev_info: #{error}"
            new_doc
          .then (doc) ->
            old_rev = doc._revs_info?[1]?.rev

If no previous version is listed (this could be the case if the document was just created, typically) then it was obviously modified and we need to restart.

            if not old_rev?
              restart_needed = true
              new_seq = seq
              return

Finally, retrieve the previous revision of the document, then use `needed` to decide whether it was modified in a way that requires a restart.

            db.get id, rev: old_rev
            .then (old_doc) ->
              restart_needed = needed config.host, old_doc, new_doc
              new_seq = seq
            .catch (error) ->
              logger.error "#{pkg.name} failed to gather data about #{new_doc._id}: #{error}"
              throw error

Return both our instance of the database and a way to cancel the process.

        {db,cancel}

    logger = require 'winston'
    second = 1000

    PouchDB = require 'pouchdb'
    pkg = require './package.json'

If this module is used as an executable, default to using the `restart-registrant` module.

    if require.main is module
      run require './restart-registrant'
    else
      module.exports = run
