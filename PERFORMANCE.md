# Angular Performance Guidance

### Сhange Detection Strategy (OnPush)
This tells Angular that the component only depends on his Inputs ( aka pure ) and needs to be checked in only the following cases:
  - The Input **reference** changes.
  - An event occurred from the component or one of his children.
  - You run change detection explicitly by calling detectChanges()/tick()/markForCheck().
    
Good example of how it works - https://stackblitz.com/github/Angular-RU/change-detection-tree/tree/onpush
Full version: https://github.com/Angular-RU/change-detection-tree
<br/> 

- **child component gets Input() object (reference is important) and change own detection**

```typescript
@Component({
...
changeDetection: ChangeDetectionStrategy.OnPush
})
...
@Input() count: { value: number };
```
- **parent component puts the same object as child expects.**
- **deep example:**

```typescript
// Parent component
@Component({
  template: `
    <tooltip [config]="config"></tooltip>
  `
})
export class AppComponent  {
  public config = {
    position: 'top'
  };
  public onClick(): void {
    this.config.position = 'bottom';
  }
}
```

```typescript
// Child component
@Component({
  selector: 'tooltip',
  template: `
    <h1>{{config.position}}</h1>
    {{runChangeDetection}}
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class TooltipComponent  {
  @Input() config;
  get runChangeDetection(): boolean {
    console.log('Checking the view');
    return true;
  }
}
```

When we click on the button we will not see any log. That’s because Angular is comparing the old value with the new value by reference, something like:

```typescript
/** Returns false in our case */
if( oldValue !== newValue ) { 
  runChangeDetection();
}
```

Just a reminder that numbers, booleans, strings, null and undefined are primitive types. All primitive types are passed by value. Objects, arrays, and functions are passed by reference.
<br/>
So in order to trigger a change detection in our component, we need to change the object reference.

```typescript
@Component({
  template: `<tooltip [config]="config"></tooltip>`
})
export class AppComponent  {
  config = {
    position: 'top'
  };
  public onClick(): void {
    this.config = {
      position: 'bottom'
    }
  }
}
```

With this change we will see that the view has been checked and the new value is displayed as expected

- **architectural example of dedication:**
<br/>

Let’s say that we have a todos component that takes a todos as input().
```typescript
@Component({
  selector: 'app-todos',
  template: `
     <div *ngFor="let todo of todos">
       {{todo.title}} - {{runChangeDetection}}
     </div>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class TodosComponent {
  @Input() todos;
  get runChangeDetection() { 
    console.log('TodosComponent - Checking the view');
    return true;
  }
}
```

```typescript
@Component({
  template: `
    <button (click)="add()">Add</button>
    <app-todos [todos]="todos"></app-todos>
  `
})
export class AppComponent {
  todos = [{ title: 'One' }, { title: 'Two' }];
  add() {
    this.todos = [...this.todos, { title: 'Three' }];
  }
}
```

The disadvantage of the above approach is that when we click on the add button Angular needs to check each todo, even if nothing has changed, so in the first click we’ll see three logs in the console.
<br/>
In the above example there is only one expression to check, but imagine a real world component with multiple bindings (ngIf, ngClass, expressions, etc.). This could get expensive.
<br/>
_We’re running change detection for no reason_
<br/>
The more performant way is to create a todo component and define its change detection strategy to be onPush. For example:

```typescript
@Component({
  selector: 'app-todos',
  template: `<app-todo [todo]="todo" *ngFor="let todo of todos"></app-todo>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class TodosComponent {
  @Input() todos;
}
@Component({
  selector: 'app-todo',
  template: `{{todo.title}} {{runChangeDetection}}`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class TodoComponent {
  @Input() todo;
  get runChangeDetection() {
    console.log('TodoComponent - Checking the view');
    return true;
  }
}
```

Now when we click the add button we’ll see a single log in the console because none of the inputs of the other todo components changed, therefore their view wasn’t checked.
<br/>
Also, by creating a dedicated component we make our code more readable and reusable.
- **also you have in the arsenal `ChangeDetectorRef` class:**

```typescript
class ChangeDetectorRef {
  markForCheck(): void
  detach(): void
  detectChanges(): void
  checkNoChanges(): void
  reattach(): void
}
```

**markForCheck** - marks all OnPush ancestors as to be checked.

```typescript
constructor(private ref: ChangeDetectorRef) {
    setInterval(() => {
      this.numberOfTicks++;
      // the following is required, otherwise the view will not be updated
      this.ref.markForCheck();
    }, 1000);
  }
```

**detach** - Detaches the change detector from the change detector tree.
<br/>
The detached change detector will not be checked until it is reattached.
<br/>
This can also be used in combination with detectChanges to implement local change detection checks.       
```typescript
constructor(private ref: ChangeDetectorRef, private dataProvider: DataProvider) {
    ref.detach();
    setInterval(() => {
      this.ref.detectChanges();
    }, 5000);
  }
```
**detectChanges** - Checks the change detector and its children.
</br>
This can also be used in combination with detach to implement local change detection checks.                  
<br/>
The following example defines a component with a large list of readonly data. Imagine, the data changes constantly, many times per second. For performance reasons, we want to check and update the list every five seconds.
<br/>
We can do that by detaching the component's change detector and doing a local change detection check every five seconds.
**checkNoChanges** - Checks the change detector and its children, and throws if any changes are detected.
<br/>
This is used in development mode to verify that running change detection doesn't introduce other changes.
**checkNoChanges** - Reattach the change detector to the change detector tree.
<br/>
This also marks OnPush ancestors as to be checked. This reattached change detector will be checked during the next change detection run.

```typescript
constructor(private ref: ChangeDetectorRef, private dataProvider: DataProvider) {}  
  set live(value) {
    if (value) {
      this.ref.reattach();
    } else {
      this.ref.detach();
    }
  }
```

<a href="https://blog.angularindepth.com/everything-you-need-to-know-about-change-detection-in-angular-8006c51d206f">More details with example</a>

### Lazy Loading

```typescript
{
  path: 'admin',
  loadChildren: 'app/admin/admin.module#AdminModule'
}
```

### Disable Zones If Possible
Big apps need to be more productive. You can control your changes with:
- OnPush
- Observables (any subject + async pipe)
- [ngModel] + (ngModelChange)
- ChangeDetectionRef 


### Tips
1. **Make Events Fast**
<br/>
Event handlers can exist in many locations within an Angular application. The most obvious examples are DOM and component event bindings. Designing these events to take as little time as possible ensures that change detection does not take more than 17 ms. If this target is not met the frame rate drops below 60 frames per second. This happens because Angular must wait for the callback to finish before change detection can continue and the update rendered.
<br/> <br/>
Let’s examine a simple example of where an event binding may take longer to execute than anticipated. The imagine code contains a component (AppComponent) that responds to a click event. It does so by passing a value from the template into a service (ListService). The service in turn utilizes that value to filter a list. Finally, control returns to the click handler before completing. It isn’t until control returns from the service that change detection can continue. As the size of the list grows, the event’s performance will degrade.

2. **Services are singletons. Restrict them**
<br/>
It means to reduce level of attainability.
<br/>
You don't need to keep in memory unnecessary information.
If service code is used only for one component - set provider to this component.

3. **Debounce/Distinct**
<br/>
Just don't forget to use, if possible
<br/>
    Use these operators (must have list):
    - debounce/debounceTime
    - throttleTime
    - distinctUntilChanged
    - share

4. **Be careful with Observables**
<br/>
Use lifecycle hook `ngOnDestroy` to unsubscribe from each subscription.

5. **Binding**
<br/>
A simple rule: don't use two-way data binding when you don't need it. <br/>
Singular binding isn't shipped out of the box, but there are several ways to implement it. 

6. **DOM**
<br/>
Remove and add the elements back instead of hiding/displaying.
<br/>
Don't call DOM directly if it's possible. When isn't - use ElementRef, @ViewChild/@ViewChildren, Renderer2

7. **Pipes**
<br/>
Avoid Computing Values in the Template
<br/>
Creating and using a custom pure pipe is generally far more convenient than restructuring the application’s data flow, but it is slightly less performant.
<br/>
A pure pipe is a pipe that behaves much like a pure function: The results of executing it are based solely on its input, and the input is left unchanged. When using a pure pipe in place of method bindings, the pipe is still executed each change detection cycle. 
<br/>
However, the execution will benefit from the fact that Angular caches the results of previous executions: If a pipe is executed more than once with the same parameters, the results of the first execution are returned.

8. **Minimize DOM Manipulations / TrackBy**
<br/>
By default, when iterating over a list of objects, Angular will use object identity to determine if items are added, removed, or rearranged. This works well for most situations. However, with the introduction of immutable practices, changes to the list’s content generates new objects. In turn, ngFor will generate a new collection of DOM elements to be rendered. If the list is long or complex enough, this will increase the time it takes the browser to render updates. To mitigate this issue, it is possible to use trackBy to indicate how a change to an entry is determined.
```typescript
import { Component } from '@angular/core';
import { ListService } from '../list.service';
@Component({
  selector: 'app-instructor-list',
  templateUrl: './instructor-list.component.html'
})
export class InstructorListComponent {
  instructorList = {name: string}[];
  constructor(private ls: ListService){
    ls.getList()
      .subscribe(list => this.instructorList = list);
  } 
  // Treat the instructor name as the unique identifier for the object
  trackByName(index, instructor) {
    return instructor.name;
  }
}
```
```html
<!--
  As the references change ngFor will continuously regenerate the DOM
  However, the presence of trackBy provides ngFor some help.
  It modifies the behavior so that it compares new data to old based on
  the return value of the supplied trackBy method.
  This allows Angular to reduce the amount of DOM update needed
-->
<ul>
  <li *ngFor="let instructor of instructorList: trackBy: trackByName" >
    <span>Instructor Name {{ instructor.name }}</span>
  </li>
</ul>
```
