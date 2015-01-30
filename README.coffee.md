A process that restarts the local registrant when needed
========================================================

    needed = require './needed'
    install = require './install'

    run = (restart) ->
      install()
      .then ({config,save}) ->
        logger.info "#{pkg.name}: Starting."
        PouchDB.debug.enable('*')
        db = new PouchDB config.provisioning

Restarting the process & saving the update sequence
---------------------------------------------------

        restart_needed = false
        new_seq = null

        save_new_seq = ->
          if new_seq?
            logger.info "#{pkg.name}: save new seq #{new_seq}."
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
          null

Especially for tests, we need to provide a way to cancel; `cancel` is given as argument to the `restart` handler, and is returned by `run`.

        cancel = ->
          logger.info "#{pkg.name}: Stopping."
          changes.cancel()
          clearInterval interval

Monitoring changes
------------------

        changes = db.changes
          since: config.update_seq ? 0
          include_docs: true
          live: true
          filter: "#{pkg.name}/global_numbers"

        .on 'change', ({id,seq,doc}) ->
          logger.info "#{pkg.name}: change on #{id}"
          new_doc = doc

We need to provide `needed` with both the previous record and the current record. Let's try to retrieve the previous record by querying for `revs_info`.

          db.get id,
            revs_info: true

If we can't access the current record for whatever reason, simply re-use the document provided by `change`.

          .catch (error) ->
            new_doc
          .then (doc) ->
            old_rev = doc._revs_info?[1]?.rev

Finally, retrieve the previous revision of the document,

            if old_rev?
              db.get id, rev: old_rev
            else

If none is specified, assumes that the document was just created (it's normal that there is no former revisions), or deleted (we weren't able to obtain `revs_info`).

              old_doc = {}
              old_doc[k] = v for own k,v of new_doc
              old_doc._deleted = not (new_doc._deleted ? false)
              old_doc

then use `needed` to decide whether it was modified in a way that requires a restart.

          .then (old_doc) ->
            if needed config.host, old_doc, new_doc
              restart_needed = true
              logger.info "#{pkg.name}: triggering restart because of #{id}"
            new_seq = seq
          .catch (error) ->
            logger.error "#{pkg.name} failed to gather data about #{new_doc._id}: #{error}"

          null

Return both our instance of the database and a way to cancel the process.

        {db,cancel}

    logger = require 'winston'
    second = 1000

    PouchDB = require 'pouchdb'
    PouchDB.debug.enable('*')
    pkg = require './package.json'

If this module is used as an executable, default to using the `restart-registrant` module.

    if require.main is module
      run require './restart-registrant'
    else
      module.exports = run
