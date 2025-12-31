---
name: writing-active-record-migrations
description: Use when authoring an Active Record migration to Discourse core or plugins.
---

# Writing Active Record Migrations

## Overview

Discourse relies on Active Record migrations to make schema and data changes to the PostgreSQL database.

### Zero Downtime Migrations

Discourse follows a blue/green deployment model which requires database migrations that introduce database schema changes to be both backward- and forward-compatible with both the blue and green versions of the application during the deploy process when migrations are executed.

To support zero-downtime migrations, Discourse runs database migrations in two phases: pre-deploy and post-deploy. Pre-deploy migrations run before the green version is deployed, while the blue version remains live. Post-deploy migrations run after the green version is live and the blue version has been fully retired. 

For Discourse core, pre-deploy migrations are stored in the `db/migrate` directory while post-deploy migrations are stored in the `db/post_migrate` directory. For plugins, migration files follow the same structure with the caveat that the `db` folder is nested under the plugin's directory.

By default, Discourse Core patches Active Record Migrations with the `Migration::SafeMigrate::SafeMigration` module to prevent unsafe SQL statements from being executed in a pre-deploy migration.

### CLI command to generate migration files 

* Core Pre-Deploy Migration: `bin/rails generate migration NAME`
* Core Post-Deploy Migration: `bin/rails generate post_migration NAME`
* Plugin Pre-Deploy Migration: `bin/rails generate plugin_migration NAME --plugin-name=PLUGIN_NAME`
* Sometimes you would like to change a site setting's name to improve clarity and precision. Generate a migration by using this generator: `bin/rails generate site_setting_rename_migration old_name new_name`

### Guidelines

#### Column Removal

1. Generate a post-deploy migration and use the following pattern to drop the column.

   ```ruby
    def up
      DROPPED_COLUMNS.each { |table, columns| Migration::ColumnDropper.execute_drop(table, columns) }
    end
   ```

2. Add `self.ignored_columns = ['name_of_column'] # TODO: Remove once <migration name> has been promoted to pre-deploy`, this will ensure that the column has been ignored by the application code prior to the actual column drop.

#### Column Rename

1. Add a pre-deploy migration which marks the existing column as read-only using `Migration::ColumnDropper.mark_readonly`, adds the new column, adds a trigger to write to new column on inserts/updates to the old column and backfills the data from the old column to the new column.
2. Update application code to read/write to new column.
3. Add a post-deploy migration to remove the old column. Note that in most cases, it is recommended to delay the removal of the old column until it has been confirmed that there has been no data loss due to the rename of the column.

#### Avoiding Application Code in Migrations

All DB migrations should avoid calling application code. For example:

```ruby
class MyMigration < ActiveRecord::Migration[6.1]
   def change
      # This is BAD
      if SiteSetting.some_setting
         add_column...
      end

      # This is BAD
      if User.happy?
         add_column...
      end
    end
end 
```

Migrations that lean on application code tend to be extremely brittle. A few years out a site setting may be removed, a user field changed, or even worse, the method name could remain but the semantics change leading to subtle migration breakages.

Instead, always lean directly on data in the database when making decisions in a migration. For example: query the `users` table directly or the `site_settings` table directly using `DB.query` and `DB.exec` from [mini_sql](https://github.com/discourse/mini_sql).

#### Adding Indexes

When adding new columns to tables, or when encountering real-world performance issues, consider adding an index to the column if it is going to be used in queries. For new columns the ActiveRecord migration helper `add_index` should work fine; for existing columns you may need to add the index concurrently in the migration to avoid lock timeout errors. 

Note that running `CREATE INDEX CONCURRENTLY` can lead to an invalid index. In other words, the index is present but marked as invalid by postgres because the statement did not complete successfully. The easiest way to ensure that a migration that creates an index concurrently is idempotent is to drop the index if it exists and then run the create index concurrently statement. Example:

```ruby
class AddImapGroupIdIndex < ActiveRecord::Migration[6.0]
  disable_ddl_transaction!

  def up
    execute <<~SQL
      DROP INDEX CONCURRENTLY IF EXISTS index_incoming_emails_on_imap_group_id
    SQL

    execute <<~SQL
      CREATE INDEX CONCURRENTLY index_incoming_emails_on_imap_group_id ON incoming_emails USING btree (imap_group_id)
    SQL
  end

  def down
    execute <<~SQL
      DROP INDEX IF EXISTS index_incoming_emails_on_imap_group_id
    SQL
  end
end
```

### Foreign Keys DB Constraints

We don’t enforce foreign keys in the database in most tables, as the overhead and complexity is not usually worth the benefit. However, there are places where adding a foreign key constraint can be useful and developers may use them if appropriate, though it may be best to discuss this with the core team beforehand.

### `NULL`able columns

Whenever possible, avoid using `NULL`able columns, even though it is the default. Every `NULL` field is a potential `nil` error waiting to happen down the line. Consider adding a default too: if a field is a boolean most of the time `false` works great here.

In some cases, especially with strings, NULLs can be sensible, examples may be “optional description” on some table, or “optional url”.

Try to limit usage of `NULL`able columns to truly optional fields.

### Testing Migrations

Whenever a migration includes data migration which has the potential to result in data loss or inaccuracy, always write an rspec test for the specific migration with test data that covers all the possible database state for the given schema. You can write an rspec test for a specific migration using the following as an example:

```ruby
# frozen_string_literal: true
# TODO: Remove before merging into `main`

require Rails.root.join(
  "plugins/chat/db/migrate/20221201024458_make_channel_slugs_unique_with_index.rb",
)

RSpec.describe MakeChannelSlugsUniqueWithIndex do
  before do
    @original_verbose = ActiveRecord::Migration.verbose
    ActiveRecord::Migration.verbose = false
  end

  after do
    ActiveRecord::Migration.verbose = @original_verbose
  end

  it "does whatever the migration is supposed to do" do
    # Fabrication of test data

    MakeChannelSlugsUniqueWithIndex.new.up

    # Assertions
  end
end
```

There are a few important points:

* You must manually include the migration file, since they are not included by default. Here it’s a plugin migration, but the same applies to core migrations
* The class name is the same one used in the migration file
* Normal fabricators and `DB.exec` code can be used here for things like removing indexes

### Updating Defaults for New Sites Only

Sometimes, for new features that may be hidden behind feature flags, we want to change the feature from being disabled by default for all sites to being disabled by default for existing sites and enabled by default for new sites. This can be done by adding a conditional which checks `Migration::Helpers.existing_site?`
