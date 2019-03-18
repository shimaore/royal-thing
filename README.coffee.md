A process that restarts the local registrant when needed
========================================================

    needed = require './needed'
    install = require './install'

    most = require 'most'

    max_delta = 200

    run = (restart,config,limit = most.never()) ->

      host = process.env.ROYAL_THING_HOST ? config.host
      db = new CouchDB config.provisioning

      debug 'Calling install()'
      {save} = await install host, db
        .catch (error) ->
          debug.dev "install failed: #{error.stack ? error}"
          throw error

      debug 'Starting.'

Restarting the process & saving the update sequence
---------------------------------------------------

      restart_needed = false
      new_seq = null

      save_new_seq = ->
        if new_seq?
          debug "Save new seq #{new_seq}."
          await save new_seq
          new_seq = null

Only restart at intervals, not on every change.

      on_interval = ->
        debug 'on_interval'
        if restart_needed
          debug "Calling `restart`."
          await restart config
          restart_needed = false
          await save_new_seq()
          debug "Restart completed."
        else
          await save_new_seq()
        return

Start the `on_interval` function.

      interval = setInterval (->
        try
          await on_interval()
        catch error
          debug.dev "Error: #{error}"
      ), config.interval ? 61*second

      limit.observe ->
        debug 'No longer monitoring'
        clearInterval interval

Force a restart if the database is too far away from our own sequence number.

      {update_seq} = await db.info()
      our_seq = (await save())?.update_seq ? 0

      debug 'Analyzing', {update_seq,our_seq}

      if update_seq isnt our_seq
        new_seq = update_seq
        restart_needed = true

      await on_interval()

Monitoring changes
------------------

      debug 'Monitoring changes'

      map_function = (emit) ->
        (doc) ->
          if doc._id.match /^number:\d+$/
            emit null

      db
      .query_changes map_function, since: our_seq, include_docs: true
      .until limit
      .observe ({id,seq,doc}) ->
        debug "Change on #{id} (seq #{seq})", doc
        new_doc = doc

We need to provide `needed` with both the previous record and the current record. Let's try to retrieve the previous record by querying for `revs_info`.

        doc = await db
          .get id,
            revs_info: true

If we can't access the current record for whatever reason, simply re-use the document provided by `change`.

          .catch (error) ->
            new_doc

        old_rev = doc._revs_info?[1]?.rev

Finally, retrieve the previous revision of the document,

        if old_rev?
          debug 'Retrieving previous version', old_rev
          old_doc = await db
            .get id, rev: old_rev
            .catch (error) ->
              debug.dev "Failed to gather data about #{id}: #{error.stack ? error}"
              null

If none is specified, assumes that the document was just created (it's normal that there is no former revisions), or deleted (we weren't able to obtain `revs_info`).

        unless old_doc?
          debug 'Constructing previous document'
          old_doc = {}
          old_doc[k] = v for own k,v of new_doc
          old_doc._deleted = not (new_doc._deleted ? false)

then use `needed` to decide whether it was modified in a way that requires a restart.

        if needed host, old_doc, new_doc
          restart_needed = true
          debug.csr "Triggering restart because of #{id}"
        new_seq = seq

        return

    debug = (require 'tangible') 'royal-thing'

    second = 1000

    CouchDB = require 'most-couchdb'

    module.exports = run
