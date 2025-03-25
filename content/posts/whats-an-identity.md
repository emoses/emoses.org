+++
title = "What's an Identity"
date = 2024-08-26
categories = ['blog']
tags = ['practices', 'databases']
draft = true
+++

## Outline:
* Row IDs identify rows in a DB
  * Important for relations between tables in a DB
  * Avoid "natural keys" for schemas
* But they're not the identity of foreign things
  * Avoid using DB rowids to refer to things with a life outside your db
* Problems
  * What if row recreated?
  * What if thing renamed
* Find rich hickey quotes
  * https://clojure.org/about/state
    * >  By identity I mean a **stable logical entity associated with a series of different values over time**
      (emph. from original)
