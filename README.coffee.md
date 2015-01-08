A process that restarts the local registrant when needed
========================================================

    needed = require './needed'
    install = require './install'

Monitor the changes in global numbers

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

      setInterval ->

Only restart at intervals, not on every change.

        if restart_needed
          restart()
          .then ->
            restart_needed = false
            save_new_seq()
        else
          save_new_seq()
      , 61*second

      db.changes
        since: config.update_seq
        include_docs: true
        live: true
        filter: "#{pkg.name}/global_numbers"
      .on 'change', ({id,seq,doc}) ->
        new_doc = doc
        if not new_doc? or new_doc._deleted
          restart_needed = true
          new_seq = seq
          return

        db.get id,
          revs_info: true
        .then ({doc}) ->
          old_rev = doc._revs_info?[1]
          if not old_rev?
            restart_needed = true
            new_seq = seq
            return

          db.get id, rev: old_rev
        .then ({old_doc}) ->
          restart_needed = needed old_doc, new_doc
          new_seq = seq
        .catch (error) ->
          logger.error "#{pkg.name} failed to gather data about #{new_doc._id}"

    logger = require 'winston'
    second = 1000
