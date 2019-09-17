# Server-side Rendering

## Pre-rendering with Angular Universal

The server-side rendering is indeed has some features, like:

* SEO (search engine optimization): At the time of writing, SPAs (single-page applications) are harder to index by search engines because the content isn’t available on load time. Therefore, the application is likely to fail on several SEO requirements.
* Initial page load could be faster: Since the application still needs to be bootstrapped after the page is loaded, there is an initial waiting time until the user can use the application. This results in a bad user experience.

Imagine the situation that we have several public static pages that are the most visited, it can be a landing page. Users can visit it from mobile devices at low speed and we don’t want to render this page every time. Solution is pre-rendering.

Server-side pre-rendering is the process of getting a finished rendered piece of page or rendering it when it is first accessed.

Our user goes to the landing page, we check if this page has already been rendered. No? Render it and save it in the appropriate file, next time we will give the rendered page just from the file without any effort.

Let's look at the below code that shows how `FilePrerenderer` class could be implemented:

```typescript
import * as fs from 'fs';
import { join } from 'path';
import { promisify } from 'util';

import { renderModuleFactory } from '@angular/platform-server';
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';

import { AppServerModuleNgFactory, LAZY_MODULE_MAP } from './dist-server/main';

const document = fs.readFileSync(join(process.cwd(), 'dist/index.html'), {
  encoding: 'utf-8'
});

const stat = promisify(fs.stat);
const readFile = promisify(fs.readFile);
const writeFile = promisify(fs.writeFile);

export class FilePrerenderer {
  /**
   * Should be stored somewhere else, stored here
   * for demonstrating purposes
   */
  private urlsToPrerender: ReadonlyArray<string> = ['/', '/about'];

  /**
   * Let's store our pre-rendered pages inside the `prerendered` folder
   */
  private path = join(process.cwd(), 'prerendered');

  constructor() {
    this.setupExitListener();
    this.createPrerenderedFolder();
  }

  prerender(url: string): Promise<string> {
    if (this.shouldSkip(url)) {
      return this.renderModuleFactory(url);
    }

    return this.getPrerendered(url);
  }

  /**
   * Determines whether URL shouldn't be pre-rendered
   */
  private shouldSkip(url: string): boolean {
    return this.urlsToPrerender.indexOf(url) === -1;
  }

  private async getPrerendered(url: string): Promise<string> {
    const path = join(this.path, `${this.transformIndexUrl(url)}.html`);

    try {
      // No exception was thrown = file exists
      await stat(path);
      // Read its content
      return readFile(path, { encoding: 'utf-8' });
    } catch {
      // File doesn't exist, let's render and save it
      const content = await this.renderModuleFactory(url);
      // Write file in background
      writeFile(path, content);
      return content;
    }
  }

  /**
   * HTML file with invalid name will be created for the `/` URL
   */
  private transformIndexUrl(url: string): string {
    return url === '/' ? 'index' : url;
  }

  private renderModuleFactory(url: string): Promise<string> {
    return renderModuleFactory(AppServerModuleNgFactory, {
      url,
      document,
      extraProviders: [provideModuleMap(LAZY_MODULE_MAP)]
    });
  }

  private createPrerenderedFolder(): void {
    if (fs.existsSync(this.path)) {
      return;
    }

    fs.mkdirSync(this.path);
  }

  private setupExitListener(): void {
    process.on('exit', () => {
      // Remove `prerendered` folder if the process exits, this means
      // that server is gonna be restarted if any developer made some changes
      // and we don't wanna receive old content
      fs.unlinkSync(this.path);
    });
  }
}
```

After all this you just need to create an instance of the class and call its method in some handler, let's look at the `Koa` example:

```typescript
const app = new Koa();
const prerenderer = new FilePrerenderer();

app.use(async (ctx: Context) => {
  ctx.body = await prerenderer.prerender(ctx.url);
});

app.listen(PORT, () => {
  console.log(`Koa server listening on http://localhost:${PORT}`);
});
```

That's it, nothing complicated.

## Pre-rendering Pages In-memory

It is also possible to use in-memory pre-rendering. This means that the content of HTML pages will not be stored in the file system, but will be permanently stored in memory, it is cheaper and much faster:

```typescript
import { join } from 'path';
import { readFileSync } from 'fs';

import { renderModuleFactory } from '@angular/platform-server';
import { provideModuleMap } from '@nguniversal/module-map-ngfactory-loader';

import { AppServerModuleNgFactory, LAZY_MODULE_MAP } from './dist-server/main';

const document = fs.readFileSync(join(process.cwd(), 'dist/index.html'), {
  encoding: 'utf-8'
});

export class InMemoryPrerenderer {
  /**
   * Should be stored somewhere else, stored here
   * for demonstrating purposes
   */
  private urlsToPrerender: ReadonlyArray<string> = ['/', '/about'];

  /**
   * Key is URL and value is HTML content
   */
  private cache = new Map<string, string>();

  prerender(url: string): Promise<string> {
    if (this.shouldSkip(url)) {
      return this.renderModuleFactory(url);
    }

    return this.getPrerendered(url);
  }

  /**
   * Determines whether URL shouldn't be prerendered
   */
  private shouldSkip(url: string): boolean {
    return this.urlsToPrerender.indexOf(url) === -1;
  }

  private async getPrerendered(url: string): Promise<string> {
    if (this.cache.has(url)) {
      return this.cache.get(url);
    }

    const content = await this.renderModuleFactory(url);
    // Save in cache
    this.cache.set(url, content);
    return content;
  }

  private renderModuleFactory(url: string): Promise<string> {
    return renderModuleFactory(AppServerModuleNgFactory, {
      url,
      document,
      extraProviders: [provideModuleMap(LAZY_MODULE_MAP)]
    });
  }
}
```

## Choosing Pre-rendering Strategy

We may have many implementations of different pre-renderers, but they all have to follow a single contract (Dependency Inversion):

```typescript
export interface Prerenderer {
  prerender(url: string): Promise<string>;
}
```

So our classes would have the below signature:

```typescript
export class FilePrerenderer implements Prerenderer {
  prerender(url: string): Promise<string> {
    // do something with file system...
  }
}

export class InMemoryPrerenderer implements Prerenderer {
  prerender(url: string): Promise<string> {
    // do something with cache
  }
}
```

We also need an enumeration and a factory function that will return the instance we need:

```typescript
const enum PrerenderStrategy {
  File,
  InMemory
}

export function createPrerenderer(strategy: PrerenderStrategy): Prerenderer {
  switch (strategy) {
    case PrerenderStrategy.File: {
      return new FilePrerenderer();
    }

    case PrerenderStrategy.InMemory: {
      return new InMemoryPrerenderer();
    }
  }
}
```

At the current time, all applications are isolated in the Docker containers, so we can specify environment variables:

```yml
// docker-compose.yml

services:
  angular-universal:
    container_name: angular-universal
    environment:
      - FILE_PRERENDER=true
      # or
      - IN_MEMORY_PRERENDER=true
```

Let's parse the `process.env`:

```typescript
function resolveStrategy(): PrerenderStrategy | never {
  const { FILE_PRERENDER, IN_MEMORY_PRERENDER } = process.env;

  if (FILE_PRERENDER) {
    return PrerenderStrategy.File;
  } else if (IN_MEMORY_PRERENDER) {
    return PrerenderStrategy.InMemory;
  }

  throw new Error(`You haven't specified what pre-renderer to use!`);
}

export function createPrerenderer(): Prerenderer {
  const strategy = resolveStrategy();

  switch (strategy) {
    case PrerenderStrategy.File: {
      return new FilePrerenderer();
    }

    case PrerenderStrategy.InMemory: {
      return new InMemoryPrerenderer();
    }
  }
}
```

All that remains is to call the `createPrerenderer` function and use its return value:

```typescript
const app = new Koa();
const prerenderer = createPrerenderer();

app.use(async (ctx: Context) => {
  ctx.body = await prerenderer.prerender(ctx.url);
});

app.listen(PORT, () => {
  console.log(`Koa server listening on http://localhost:${PORT}`);
});
```
