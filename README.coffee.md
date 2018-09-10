A process that restarts the local registrant when needed
========================================================

    needed = require './needed'
    install = require './install'

    max_delta = 200

    run = (restart,config) ->

      debug 'Calling install()'
      {config,save} = await install config
        .catch (error) ->
          debug "install failed: #{error.stack ? error}"
          {}

      debug "Starting."
      db = new PouchDB config.provisioning

Restarting the process & saving the update sequence
---------------------------------------------------

      restart_needed = false
      new_seq = null

FIXME: Save the new seq in the database' _local vars, not in the config.

      save_new_seq = ->
        if new_seq?
          debug "Save new seq #{new_seq}."
          await save new_seq
          new_seq = null

Only restart at intervals, not on every change.

      on_interval = ->
        # debug 'on_interval'
        if restart_needed
          debug "Calling `restart`."
          await restart config, cancel
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
          debug "Error: #{error}"
      ), config.interval ? 61*second

Force a restart if the database is too far away from our own sequence number.

      {update_seq} = await db.info()
      our_seq = config.update_seq ? 0

Database was reset, or other inconsistency where the database ends up being "behind" us:

      if update_seq < our_seq
        new_seq = update_seq
        restart_needed = true

Too many changes to reasonnably process:

      if update_seq > our_seq + (config.max_delta ? max_delta)
        new_seq = update_seq
        restart_needed = true

      await on_interval()

Especially for tests, we need to provide a way to cancel; `cancel` is given as argument to the `restart` handler, and is returned by `run`.

      cancel = ->
        debug "Stopping."
        changes?.cancel()
        clearInterval interval
        db = null

Monitoring changes
------------------

      changes = db.changes
        since: config.update_seq ? 0
        include_docs: true
        live: true
        filter: "#{pkg.name}/global_numbers"

      .on 'error', (error) ->
        debug "Change error #{error}"
        cancel?()
        debug "Change error #{error} → restart"
        run restart, config
        return

      .on 'uptodate', ->
        debug "Change up-to-date"

      .on 'change', ({id,seq,doc}) ->
        debug "Change on #{id} (seq #{seq})"
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
          old_doc = await db
            .get id, rev: old_rev
            .catch (error) ->
              debug "Failed to gather data about #{id}: #{error.stack ? error}"
              null

If none is specified, assumes that the document was just created (it's normal that there is no former revisions), or deleted (we weren't able to obtain `revs_info`).

        unless old_doc?
          old_doc = {}
          old_doc[k] = v for own k,v of new_doc
          old_doc._deleted = not (new_doc._deleted ? false)

then use `needed` to decide whether it was modified in a way that requires a restart.

        if needed (process.env.ROYAL_THING_HOST ? config.host), old_doc, new_doc
          restart_needed = true
          debug "Triggering restart because of #{id}"
        new_seq = seq

        return

Return both our instance of the database and a way to cancel the process.

      debug 'Ready'

      {db,cancel}

    pkg = require './package.json'
    debug = (require 'debug') pkg.name

    second = 1000

    PouchDB = require 'ccnq4-pouchdb'

    module.exports = run
