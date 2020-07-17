---
layout: post
title: "Multiple databases"
subtitle: "using ActiveRecord"
date: 2020-07-05 00:30:00 -0400
background: '/img/posts/09.jpg'
---

and wrapped it in:
```
ActiveRecord::Base.connected_to(role: :writing) do
  <code>
end
```
ActiveRecord is the Rails ORM that we use to manage our connection to the database
we're using an ActiveRecord method called connected_to, which specifies which "role" we want to use
we have two ActiveRecord roles - "writing" and "reading":
```
//app/models/application_record.rb (config file, defining how to access records)
  connects_to database: { writing: :primary, reading: :replica } if ENV.fetch('IS_MULTI_DB', 'false') == 'true'
```
that says that the writing role connects to the primary DB, and the reading role connects to the secondary DB
```
// config/database.yml
multi-db: &multi-db
  primary:
    <<: *container
    host: '<%= ENV["DATABASE_HOST"] %>'
  replica:
    <<: *container
    host: '<%= ENV["DATABASE_HOST_READ_ONLY"] %>'
    replica: true
```
does that make sense?