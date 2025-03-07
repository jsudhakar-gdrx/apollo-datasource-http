# Apollo HTTP Data Source

[![CI](https://github.com/StarpTech/apollo-datasource-http/actions/workflows/ci.yml/badge.svg)](https://github.com/StarpTech/apollo-datasource-http/actions/workflows/ci.yml)

Optimized JSON HTTP Data Source for Apollo Server

- Uses [Undici](https://github.com/nodejs/undici) under the hood. It's around `60%` faster than `apollo-datasource-rest`
- Request Deduplication (LRU), Request Cache (TTL) and `stale-if-error` Cache (TTL)
- Support [AbortController ](https://github.com/mysticatea/abort-controller) to manually cancel all running requests
- Support for [Apollo Cache Storage backend](https://www.apollographql.com/docs/apollo-server/data/data-sources/#using-memcachedredis-as-a-cache-storage-backend)

## Documentation

View the [Apollo Server documentation for data sources](https://www.apollographql.com/docs/apollo-server/features/data-sources/) for more details.

## Usage

To get started, install the `apollo-datasource-http` package:

```bash
npm install apollo-datasource-http
```

To define a data source, extend the [`HTTPDataSource`](./src/http-data-source.ts) class and implement the data fetching methods that your resolvers require. Data sources can then be provided via the `dataSources` property to the `ApolloServer` constructor, as demonstrated in the section below.

```ts
// instantiate a pool outside of your hotpath
const baseURL = 'https://movies-api.example.com'
const pool = new Pool(baseURL)

const server = new ApolloServer({
  typeDefs,
  resolvers,
  dataSources: () => {
    return {
      moviesAPI: new MoviesAPI(baseURL, pool),
    }
  },
})
```

Your implementation of these methods can call on convenience methods built into the [HTTPDataSource](./src/http-data-source.ts) class to perform HTTP requests, while making it easy to pass different options and handle errors.

```ts
import { Pool } from 'undici'
import { HTTPDataSource } from 'apollo-datasource-http'

const datasource = new (class MoviesAPI extends HTTPDataSource {
  constructor(baseURL: string, pool: Pool) {
    // global client options
    super(baseURL, {
      pool,
      clientOptions: {
        bodyTimeout: 5000,
        headersTimeout: 2000,
      },
      requestOptions: {
        headers: {
          'X-Client': 'client',
        },
      },
    })
  }

  onCacheKeyCalculation(request: Request): string {
    // return different key based on request options
  }

  async onRequest(request: Request): Promise<void> {
    // manipulate request before it is send
    // for example assign a AbortController signal to all requests and abort

    request.signal = this.context.abortController.signal

    setTimeout(() => {
      this.context.abortController.abort()
    }, 3000).unref()
  }

  onResponse<TResult = unknown>(request: Request, response: Response<TResult>): Response<TResult> {
    // manipulate response or handle unsuccessful response in a different way
    return super.onResponse(request, response)
  }

  onError(error: Error, request: Request): void {
    // in case of a request error
    if (error instanceof RequestError) {
      console.log(error.request, error.response)
    }
  }

  async createMovie() {
    return this.post('/movies', {
      body: {
        name: 'Dude Where\'s My Car',
      }
    })
  }

  async getMovie(id) {
    return this.get(`/movies/${id}`, {
      query: {
        a: 1,
      },
      context: {
        tracingName: 'getMovie',
      },
      headers: {
        'X-Foo': 'bar',
      },
      requestCache: {
        maxTtl: 10 * 60, // 10min, will respond for 10min with the cached result (updated every 10min)
        maxTtlIfError: 30 * 60, // 30min, will respond with the cached response in case of an error (for further 20min)
      },
    })
  }
})()
```

## Hooks

- `onCacheKeyCalculation` - Returns the cache key for request memoization.
- `onRequest` - Is executed before a request is made. This can be used to intercept requests (setting header, timeouts ...).
- `onResponse` - Is executed when a response has been received. This can be used to alter the response before it is passed to caller or to log errors.
- `onError` - Is executed for any request error.

## Error handling

The http client throws for unsuccessful responses (statusCode >= 400). In case of an request error `onError` is executed. By default the error is rethrown as a `ApolloError` to avoid exposing sensible information.

## Benchmark

See [README.md](benchmarks/README.md)

## Production checklist

This setup is in use with Redis. If you use Redis ensure that limits are set:

```
maxmemory 10mb
maxmemory-policy allkeys-lru
```

This will limit the cache to 10MB and removes the least recently used keys from the cache.

## Versioning

We follow [semver](https://semver.org/). Major version zero (0.y.z) is for initial development. Anything MAY change at any time. The public API **SHOULD NOT** be considered stable ([source](https://semver.org/#spec-item-4)).

## Node.js support

We test this software against **latest** major releases of the [Node.js LTS policy](https://github.com/nodejs/Release). `Current` is included to catch regression earlier.
