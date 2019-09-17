# Routing

## Resolvers

Don't load any data during view construction. Angular already has this feature out of the box, even AngularJS had `$routeProvider` where you could declare the `resolve` property.

Resolvers are used for performing some asyncronous work before the component is rendered. If Angular gives us such possibility, should we omit it?

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-todos',
  template: `
    <app-todo *ngFor="let todo of todos" [todo]="todo"></app-todo>
  `
})
export class TodosComponent {
  todos: Todo[] = [];

  constructor(private todoService: TodoService) {
    this.todoService.getTodos().subscribe(todos => {
      this.todos = todos;
    });
  }
}
```

Such approach is considered as a bad practice as you force Angular to do multiple re-renders. That's why Google integrated resolvers for us. You should be able to access data to render already in the constructor:

```typescript
import { Component } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
 
@Component({
  selector: 'app-todos',
  template: `
    <app-todo *ngFor="let todo of todos" [todo]="todo"></app-todo>
  `
})
export class TodosComponent {
  todos: Todo[] = this.route.snapshot.data.todos;
 
  constructor(private route: ActivatedRoute) {}
}
```

And the resolver will look like this:

```typescript
@Injectable()
export class TodosResolver implements Resolve<Todo[]> {
  constructor(private todoService: TodoService) {}
 
  resolve(): Observable<Todo[]> {
    return this.todoService.getTodos();
  }
}
```

Don't forget to configure the **resolve** property:

```typescript
const routes: Routes = [
  {
    path: 'todos',
    component: TodosComponent,
    resolve: {
      todos: TodosResolver
    }
  }
];
```

If you need to send some HTTP request on a button click - sure go for it. I'm talking about data preloading.

Resolvers can also be used for caching, resolvers are singleton classes, as opposed to the components that have a lifecycle `constructor => ngOnDestroy`:

```typescript
@Injectable()
export class TodosResolver implements Resolve<Todo[]> {
  private todos: Todo[] = [];

  constructor(private todoService: TodoService) {}

  resolve(): Todo[] | Observable<Todo[]> {
    if (this.todos.length) {
      return this.todos;
    }

    return this.todoService.getTodos().pipe(
      tap(todos => {
        this.todos = todos;
      })
    );
  }
}
```
