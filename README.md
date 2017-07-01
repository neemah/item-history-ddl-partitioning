## Item with history

create extension if not exists plv8;

create table if not exists _created (
    created     timestamptz not null default now()
);

create table if not exists _updated (
    updated     timestamptz not null default now()
);

create table if not exists _timestampable (
) inherits (_created, _updated);

create table if not exists _archivable (
    archived    boolean NOT NULL DEFAULT false
);

create table if not exists some_table (
  id  serial not null,
  name varchar (1024) not null,
  primary key(id)
);

insert into some_table (name) values ('helo der');

create table if not exists item_skel (
    id                          int not null,
    title                       varchar(1024) not null,
    version                     int not null default 1,
    some_external_id            int not null -- NO REFERENCE KEY HERE!
) inherits (_timestampable, _archivable);

create table if not exists item_history (
    some_external_id            int not null references "some_table" on delete restrict,
    primary key(id, version)     -- !!!!
) inherits (item_skel);

create index item_history__id_idx on card_history (id);

create table if not exists item (
    id                          serial PRIMARY KEY not null,
    some_external_id            int not null references "some_table" on delete restrict
) inherits (item_skel);

create or replace function item_insert(fieldz jsonb) RETURNS json AS
$$
    var __parseJSONB = function (data) {
        if (typeof data === 'object') {
            return data;
        }

        return JSON.parse(data);
    }

    var result = {};

    var fields = __parseJSONB(fieldz);

    plv8.subtransaction(function () {
        var columns = [];
        var vals = [];
        var params = [];

        var i = 1;

        Object.keys(fields).forEach(function (key) {
            columns.push(plv8.quote_ident(key));
            vals.push('$' + i++);
            params.push(fields[key]);
        });

        var query = 'insert into item (' + columns.join(', ') + ') values (' + vals.join(', ') + ') returning *';

        result = plv8.execute(query, params)[0];
      })

    return result;
$$
language plv8;

create or replace function item_update(id int, fieldz jsonb) RETURNS json AS
$$
  var __parseJSONB = function (data) {
      if (typeof data === 'object') {
          return data;
      }

      return JSON.parse(data);
  }

  var fields = __parseJSONB(fieldz);

  var result = {
    item: {},
    old: {}
  };

  plv8.subtransaction(function () {
    // "for update" is required to obtain exclusive lock!
    var old = plv8.execute('select * from item where id = $1 for update;', [id])[0];

    var j = 1;
    var historyColumns = Object.keys(old);

    var historyInsertSql = 'insert into item_history (' +
      historyColumns.map(function (col) {return plv8.quote_ident(col); }).join(', ') +
    ') values (' +
      historyColumns.map(function () {return '$' + j++; }).join(', ') +
    ')';

    plv8.execute(historyInsertSql, historyColumns.map(function (col) {return old[col];}));

    var setPart = [];
    var params = [id];

    var i = 2;

    Object.keys(fields).forEach(function (col) {
      setPart.push(plv8.quote_ident(col) + ' = $' +  i++);
      params.push(fields[col]);
    });

    // version update

    setPart.push('"version" = $' + i++);
    params.push(old.version + 1);

    var query = 'update item set ' + setPart.join(', ') + ' where id = $1 returning *';

    result.item = plv8.execute(query, params)[0];
    result.old = old;
  });

  return result;
$$
language plv8;


select item_insert('{"title":"abracadabra", "some_external_id": 1}');
select item_update(1, '{"title":"cadabrararar"}');

select item_update(1, '{"title":"sdlkafjskdlf"}');


-- magic starts here

-- all data from item_history and item with 1 magic query:
select * from item_skel;

-- because check this:
explain analyze select * from item_skel;

-- item_skel actually has 0 rows

-- only history items:
select * from item_history where id = 1;


-- actual version:
select * from item where id = 1;
