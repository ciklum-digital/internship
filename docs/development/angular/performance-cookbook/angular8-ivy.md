# Angular 8+ (Ivy)

## Asynchronous Components

Ivy is a compiler that provides a completely new pipeline.

This compiler also allows us to load components asynchronously on demand and this component is not linked with any module.

Let's imagine that we have some blog, there are 100 articles. When you click on the article it reveals and you see the full content. Let's create our asynchronous `ArticleComponent`:

```typescript
@Component({
  selector: 'app-article',
  template: `
    <div [innerHTML]="content"></div>
  `,
  styleUrls: ['./article.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ArticleComponent {
  content: SafeHtml | null = null;

  constructor(private sanitizer: DomSanitizer) {}

  setContent(content: string): void {
    this.content = this.sanitizer.bypassSecurityTrustHtml(content);
  }
}
```

How can we load and render it asynchronously? Very simply! The dynamic import is our weapon:

```typescript
import {
  Component,
  ɵrenderComponent as renderComponent,
  Injector,
  ChangeDetectionStrategy,
  OnDestroy,
  ɵmarkDirty as markDirty
} from '@angular/core';

@Component({
  selector: 'app-articles',
  template: `
    <div *ngFor="let article of articles; index as index">
      ...
      <button (click)="openArticle(article, index)">Open article</button>
      <app-article class="article-{{ index }}"></app-article>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ArticlesComponent implements OnDestroy {
  articles: Article[] = [];

  /**
   * Dynamic import can also be used as a type definition for asynchronous modules
   */
  private articleComponent: import('./article.component').ArticleComponent | null = null;

  constructor(private injector: Injector) {}

  ngOnDestroy(): void {
    this.destroyArticleComponent();
  }

  async openArticle({ content }: Article, index: number): Promise<void> {
    this.destroyArticleComponent();

    const { ArticleComponent } = await import('./article.component');

    this.articleComponent = renderComponent(ArticleComponent, {
      host: `.article-${index}`,
      injector: this.injector
    });

    this.articleComponent.setContent(content);
    markDirty(this.articleComponent);
  }

  private destroyArticleComponent(): void {
    if (this.articleComponent) {
      this.articleComponent.dispose();
      this.articleComponent = null;
    }
  }
}
```

`ɵrenderComponent` is available to use right now, NOTICE this function works only when the Ivy compiler is enabled. You also need to manually control the life cycle of this component and re-render it yourself. `markDirty` does the same thing as `markForCheck` but it also schedules change detection via `requestAnimationFrame`.
