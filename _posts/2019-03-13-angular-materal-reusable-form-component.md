---
title: "Angular: experimenting with reusable form components"
categories:
  - Web
tags:
  - angular
  - material
---

{% include figure image_path="https://farm9.staticflickr.com/8137/8705974432_a617300057_h.jpg" alt="Red Blocks by Ram Yoga" caption="'Red Blocks' by [Ram Yoga](https://www.flickr.com/photos/ramyoga/8705974432/)" %}

> **TL;DR** This article describes two possible ways of creating and managing reusable components regrouping several inputs in an Angular form.
> First method is to handle only the visual template of the reusable component and let the parent component fully control the child form group, which belongs to the reusable component.
> Second method is to encapsulate component's logic and add a child form group to a parent dynamically.

This article is a result of my experiments with creating reusable form group child components.

## Why bother

Recently I faced a problem of creating reusable components to homogenize the behavior and UX of an Angular application. So the goal is to have is a set of simple pre-configured general-purpose form components. When creating a new form with trivial behaviors one can choose from the pre-configured components and construct the required form.

The application in question is written using Angular and [Material](https://material.angular.io/) components, so the reusable components would use Material components. The application uses [ReactiveForms](https://angular.io/guide/reactive-forms) to handle user input.

## Experiments

After looking for a while I found some articles about reusable form component. You can find them listed in References & Credits section.
Having all these information I was curious to play a bit with proposed techniques and to find out for myself.


### Approach 1 - component as a template. String coupling.

The first approach of creating a reusable component is to use its template part only. No logic encapsulation in the reusable component itself. Only the configuration of the view (template).

In that case a parent component has to have a full control of its children and manages all aspects of it's children's' life cycle - creation, assigning validators, listening to events.

For this approach to work, a parent must knows what input controls are used in a child component to be able to bind to them, and a child must have  access to its parent.
This is achieved with two things:

* reusable component has no binding to its template inputs, the parent has
* reusable component has a reference to its parent form group, this way child form group is seamlessly integrated in a parent.

Let's look at some code examples to make it more clear.
Stackblitz of the example can be found [here](https://stackblitz.com/edit/angular-j7thpy).

Here is an extract of the parent component code:

_app.component.ts_
```ts
export class AppComponent  {
  ...
  public simpleForm: FormGroup;

  constructor(private fb: FormBuilder) {
    this.simpleForm = fb.group({
      // parent's own input
      name: [''],
      // child's component input control, control name is passed via @Input to the child 
      selectCtrl: ['', Validator.required]
    });
    // parent tracks child's input state change
    this.simpleForm.controls.selectCtrl.valueChanges.subscribe( value =>
      {
        this.selectedOption = value;
      });
  }
}
```

The parent component AppComponent holds the root form group `simpleForm`. The parent creates form bindings for all input controls in a form:
* the `selectCtrl` control - it belongs to the SimpleSelect component which we will see later.
* the `name` control - it belongs to AppComponent itself

The same way the parent component assigns 'required' validator and a listener for child's `selectCtrl` control.
When the value of a select is changed it is shown on the page.

Let's see how the parent lets its child know that he is a part of a bigger picture.

_app.component.html_
```html
<form [formGroup]="simpleForm">
  <input matInput formControlName="name"/>
  <!-- Passing parent form, options list and control name to use -->
  <simple-select [parentForm]="simpleForm" formInnerControlName="selectCtrl"
                [options]="options">
  </simple-select>
</form>
<div>Option selected: {{selectedOption?.name}}</div>
```

And here is a child component.

_simple-select.component.ts_
```ts
export class SimpleSelectComponent {
  /**
   * List of options to use
   */
   @Input() options: SimpleOption[];

  /**
   * Parent FormGroup for inputs of this component
   */
  @Input() parentForm: FormGroup;

  /**
   * mat-select control name
   */
  @Input() formInnerControlName: string;

}
```

The name for the select input control is passed as component input named `formInnerControlName`. This way the parent can bind to the inner select control.

And finally here is a template of the reusable component example.

_simple-select.component.html_
```html
<form [formGroup]="parentForm">
<mat-form-field>
  <mat-select placeholder="Simple Select" formControlName="{{formInnerControlName}}">
    <mat-option *ngFor="let opt of options" [value]="opt">
      {{opt.name}}
    </mat-option>
  </mat-select>
  <mat-error *ngIf="parentForm.get(formInnerControlName).hasError('required')">
      This field is required.
  </mat-error>
</mat-form-field>
</form>
```

It's a select input with 'required' input error message handling. If the control bind to the select has a 'required' validator configured, error handling will be activated. In case of no validator, well, the input is always valid. 

![MAGIC](https://media.giphy.com/media/12NUbkX6p4xOO4/giphy.gif)

But feels a bit weird.
The parent has to know too much about the insides of the child component it uses.

So let's move forward.

### Autonomous reusable component

This approach consists of encapsulating all the logic inside the reusable component.
This allows to simplify the development of the form as it is composed of independent bricks. 

In the example below, which was inspired by  Jeroen Bastijns's article, a reusable inputs component knows nothing about its parent. It just needs to let the parent know about its existence. We do that by emitting a child FromGroup as soon as component is ready.

Stackblitz of the example can be found [here](https://stackblitz.com/edit/angular-xxhs6b).

_simple-select.component.ts_
```ts
export class SimpleSelectComponent implements OnInit {
 ...
  /**
   * Emits a message containing this component FormGroup.
   * The parent component can simply add the form group as a child.
   */
  @Output() onComponentReady: EventEmitter<FormGroup> = new EventEmitter<FormGroup>();

  private componentFormGroup: FormGroup;
  
  constructor(private fb: FormBuilder) { }

  ngOnInit() {
    ...
    this.onComponentReady.emit(this.componentFormGroup);
  }

}
```

The parent subscribes to 'onComponentReady' event and adds a child to its root FormGroup.

_app.component.ts_
```ts
export class AppComponent  {
  public simpleForm: FormGroup;
  ...

  constructor(private fb: FormBuilder){
    this.simpleForm = fb.group({
      name: ['']
    });
  }

  public addChild(childName:string, childGroup: FormGroup) {
    this.simpleForm.addControl(childName, childGroup);
  }

}
```

And here is a template of the parent component.

_app.component.html_
```html
<form [formGroup]="simpleForm">
  <input placeholder="Name" matInput formControlName="name"/>
  <simple-select (onComponentReady)="addChild('selectGroup', $event)" 
                (selectChanged)="showOption($event)"
                [options]="options" required notEmpty>
  </simple-select>
</form>
...
```

So the reusable 'simple-select' form component encapsulates all the logic of handling its inputs, the select in this case. Notice 'required' and 'notEmpty' input properties of the component which are used to configure the component behavior.

_simple-select.component.ts_
```ts
export class SimpleSelectComponent implements OnInit {
  ...

  @Input()
  get required() { return this._required; }
  set required(value: any) { this._required = coerceBooleanProperty(value); }
  
  @Input()
  get notEmpy() { return this._notEmpy; }
  set notEmpy(value: any) { this._notEmpy = coerceBooleanProperty(value); }

  ...

  constructor(private fb: FormBuilder) { }

  ngOnInit() {
    // construct validators based on component input properties
    const validators = [];
    if (this.required) {
      validators.push(Validators.required)
    }
    this.componentFormGroup = this.fb.group({
      select: ['', validators]
    });
    this.componentFormGroup.controls.select.valueChanges.subscribe(
      value => this.selectChanged.emit(value)
    );
    this.onComponentReady.emit(this.componentFormGroup);
  }

}
```

The template of the reusable component.

_simple-select.component.html_
```html
<form [formGroup]="componentFormGroup">
<mat-form-field>
  <mat-select placeholder="Simple Select" formControlName="select">
    <mat-option *ngFor="let opt of options" [value]="opt">
      {{opt.name}}
    </mat-option>
  </mat-select>
  <mat-error *ngIf="required && componentFormGroup.controls.select.hasError('required')">
      This field is required.
  </mat-error>
  <mat-error *ngIf="checkEmpty && !(options || options.length)">
       Option list is empty.
  </mat-error>
</mat-form-field>
</form>
```

And that' it. Nice and neat.

Personally I prefer this approach as it tends to be what a good software design is: hight cohertion and loose coupling.


# Conclusion

We looked at two approaches of creating reusable form components: 
1. only template is reusable
2. encapsulated form component

The first approach is very easy to set up, but tends to be more complex to use since the parent component must know everything about the inside structure of its child to be able to use it properly.

The second approach offers neater and more understandable integration. Which results in simpler usage. However this approach need more effort ad it shifts the responsibility to the child component to provide the clean configuration interface and to handle its resources properly.

# References & Credits
1. [Angular reactive forms guide](https://angular.io/guide/reactive-forms)
2. Article by Jeroen Bastijns [Angular: Reusable form components](https://medium.com/@bastijns.jeroen/angular-reusable-form-components-f9233193138c)
3. [Dynamic Nested Reactive Forms In Angular]() https://medium.com/@joshblf/dynamic-nested-reactive-forms-in-angular-654c1d4a769a)