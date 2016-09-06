Title: EmberJS - Customizing DS.RESTSerializer
Date:2016-03-13
Tags: emberjs, ember-data, restserializer
Category: emberjs
Authors: José San Gil

(Updated 2016-04-03: PR merged)

Two weeks ago I gave a talk at [EmberJS BCN Meetup](http://www.meetup.com/Ember-js-Barcelona/) on ["How to use Ember Data with your REST API"](). It was my first talk and fortunately it didn't go bad. I wanted to explain how can you customize Ember Data to work with APIs that don't comply with [JSON API spec](http://jsonapi.org/), which is the default Response format expected by `Ember Data` since version 2.0.

However, while I was livecoding a custom `DS.RESTSerializer` as a demo, I realized that `normalizeHash` property wasn't listed on the API docs anymore (Ember Data 2.4 docs), it was referenced within the `normalize` method definition, though.

Last week I started digging into this topic and realized that it was **deprecated** on [Ember Data 1.13 release](https://github.com/emberjs/data/blob/master/CHANGELOG.md#release-1135-july-8-2015). However, docs were only partially updated and `normalizeHash` didn't appear anymore as property of `DS.RESTSerializer`, but the `normalize` method still had references:

[Ember Data 2.4 docs](https://github.com/emberjs/data/blob/v2.4.0/addon/serializers/rest.js#L91):
>  If you want to do normalizations specific to some part of the payload, you can specify those under `normalizeHash`.

>  For example, if the `IDs` under `"comments"` are provided as `_id` instead of `id`, you can specify how to normalize just the comments:

```javascript
  //app/serializers/post.js
  import DS from 'ember-data';
  export default DS.RESTSerializer.extend({
    normalizeHash: {
      comments: function(hash) {
        hash.id = hash._id;
        delete hash._id;
        return hash;
      }
    }
  });
```

Then, I found an [issue](https://github.com/emberjs/data/issues/4186) in Ember Data's repo asking to solve this. I opened a [PR](https://github.com/emberjs/data/pull/4228) that fixes it (already merged). Also, a deprecation warning has been added by [PR 4258](https://github.com/emberjs/data/pull/4258).

## Munging the payload
So, what's the right way to use RESTSerializer?
If you need to do general manipulations to your API's response, start with the `normalizeResponse` method. Let's say that our non-JSON API Football Teams API has the following endpoint/response:

```GET /api/teams```
```json
{
  "result": "Ok",
  "data": {
    "teams": [
      {
        "name": "Leicester City F.C.",
        "league": "Premier League",
        "points": 63,
        "won": 18,
        "drawn": 9,
        "lost": 9,
        "gf": 53,
        "ga": 31,
        "founded": 1884,
        "arena": "King Power Stadium"
      },
      {
        "name": "Totteham Hotspur F.C.",
        "league": "Premier League",
        "points": 58,
        "won": 16,
        "drawn": 10,
        "lost": 4,
        "gf": 53,
        "ga": 24,
        "founded": 1882,
        "arena": "White Hart Lane"
      },
      {
        "name": "Arsenal F.C.",
        "league": "Premier League",
        "points": 52,
        "won": 15,
        "drawn": 7,
        "lost": 7,
        "gf": 46,
        "ga": 30,
        "founded": 1886,
        "arena": "Emirates Stadium"
      },
      {
        "name": "Manchester City F.C.",
        "league": "Premier League",
        "points": 51,
        "won": 15,
        "drawn": 6,
        "lost": 8,
        "gf": 52,
        "ga": 31,
        "founded": 1880,
        "arena": "City of Manchester Stadium"
      }
    ]
  },
  "links": {
    "prev": "http://api.footballteams/resources/teams/?page=1&per_page=4",
    "next": "http://api.footballteams/resources/teams/?page=2&per_page=4",
    "first": "http://api.footballteams/resources/teams/?page=1&per_page=4",
    "last": "http://api.footballteams/resources/teams/?page=5&per_page=4",
  }
}
```

Then, we have defined `team` model as follows:

```javascript
// app/models/team.js
import DS from 'ember-data';

const { attr } = DS;

export default DS.Model.extend({
  name: attr('string'),
  league: attr('string'),
  points: attr('number'),
  won: attr('number'),
  drawn: attr('number'),
  lost: attr('number'),
  goalsAgainst: attr('number'),
  goalsFor: attr('number'),
  founded: attr('number'),
  stadium: attr('string')
});

```

Let's write our custom `normalizeResponse` method to get rid of some useless attributes:

```javascript
// app/serializers/team.js
import DS from 'ember-data';

export default DS.RESTSerializer.extend({
  primaryKey: 'name',
  
  normalizeResponse(store, model, payload, id, requestType) {
    // deletes useless 'result' attribute
    delete payload.result;
    // removes data attribute and keep teams
    payload.teams = payload.data.teams;
    delete payload.data;
    
    // looks like this endpoint requires pagination
    payload.links = doSomethingToHandlePagination();
    delete payload.links

    return this._super(...arguments);
  }

});
```
API Response does not include an id for each `team`, so we are setting `name` as an identifier for each record. Nevertheless, using the team's `name` as `id` is probably an anti-pattern; you could try to generate an id for each record in the serializer, but that's beyond the scope of this post.


## Important
As we're using `normalizeResponse`, this normalization will apply to any request related to the `team` model. If your API structure isn't consistent and varies for each endpoint (`GET api/teams/{team_id}`, `POST api/teams`, `PUT api/teams/{team_id}`, etc), you should try with `normalizeFindAllResponse`, `normalizeCreateRecordResponse` or any `normalize*` method that matches your API request. For example, the code above could have been written using the `normalizeFindAllResponse` method as follows:

```javascript
// app/serializers/team.js
import DS from 'ember-data';

export default DS.RESTSerializer.extend({
  primaryKey: 'name',
  
  normalizeResponse(store, model, payload, id, requestType) {
    // deletes useless 'result' attribute
    delete payload.result;
    return this._super(...arguments);
  },

  normalizeFindAllResponse(store, model, payload, id, requestType) {
    // removes data attribute and keep teams
    payload.teams = payload.data.teams;
    delete payload.data;
    
    // looks like this endpoint requires pagination
    payload.links = doSomethingToHandlePagination();
    delete payload.links
    return this._super(...arguments);
  },
});
```

Our payload it's almost normalized, but we still need to match `ga`, `gf`, `arena` attributes with the `team` model attributes `goalsAgainst`, `goalsFor` and `stadium`, respectively. Let's add a custom `normalize` method to our `team` serializer:

```javascript
// app/serializers/team.js
import DS from 'ember-data';

export default DS.RESTSerializer.extend({
  
  normalizeResponse(store, model, payload, id, requestType) {
    // ... as defined above
  },

  normalize(model, hash, prop) {
    hash.goalsAgainst = hash.ga;
    delete hash.ga;
    hash.goalsFor = hash.gf;
    delete hash.gf;
    hash.stadium = hash.arena;
    delete hash.arena;
    return this._super(model, hash, prop);
  },
});
```
The `normalize` method will be called for each `team` within the API response. Now, the payload fully matches what's expected by Ember Data and the `team` model will get populated.  

*The methods described above work for Ember Data 1.13 - 2.4*.

The slides are available [here](https://github.com/jsangilve/ember-bcn-meetup-march-2016/blob/master/ember-meetup.pdf).

