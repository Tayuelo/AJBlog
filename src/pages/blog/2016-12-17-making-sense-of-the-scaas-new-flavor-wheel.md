---
templateKey: blog-post
title: Angular Signals for Beginners
date: 2023-12-04T16:47:27.733Z
description: A new Angular feature that will improve our apps performance and
  developer experience!
featuredpost: false
featuredimage: /img/angular-portrait.jpeg
tags:
  - signals
  - angular
---
![Angular 17](/img/angular-portrait.jpeg "Angular 17 Logo")

- - -

This has been a great year for the Angular team and developers, with two major releases unfolding a bunch of new features and improvements to the framework, the Angular Renaissance is really raising good expectations for the future of web development and today I want to stress some of the most popular features introduced in [Angular V16](https://blog.angular.io/angular-v16-is-here-4d7a28ec680d)

### Signals

Let’s start by definitions: A Signal is a wrapper function that allows you to define values that notify its consumers whenever a new value is emitted, making them reactive. This signal can be a dependency of another signal, effect, or even a template.

#### Writable signals

You can define a new signal by using the signal function and providing an initial value.

const age = signal(10);

console.log(age()); // 10

Note the use of parentheses, it is meant to read the signal by calling its getter function, if you don’t add the parentheses, you’ll be accessing only the reference to the signal, and not its value.

In order to change the value of a Writable Signal, you need to use its set function in this way:

```typescript
age.set(20);

console.log(age()); // 20
```

This is useful for changing primitive values or replacing data structures when the new value is independent of the old one.

When we need to change the value of the signal considering the previous value, we use update:

```
age.update(prev => ++prev);

console.log(age()); // 21
```

Writable signals implement the WritableSignal interface.

#### Computed signals

Computed signals derive their values from other signals and can’t be modified because they’re readonly signals:

```
const price = signal(30);

const priceWithDiscount = computed(() => price() * 0.5);

console.log(priceWithDiscount()); // 15

priceWithDiscount.set(10); // property 'set' does not exist in type Signal<T>
```

The computed signal priceWithDiscount has a dependency, which is price, that means that whenever price updates, Angular knows that priceWithDiscount needs to be updated as well.

The function `() => price() * 0.5` will not be executed until you read priceWithDiscount’s getter

Signals values are memoized, this means that every new read of a signal will return its current cached value, only the first read will trigger any calculations.

Here’s an example:

```
const users = signal(\['Emma', 'Laura', 'Linda', 'Adan']);

const letter = signal('L');

const filteredUsers = computed(() =>

 users().filter((user) => user.startsWith(letter()));

console.log(filteredUsers()); // \['Laura', 'Linda'] <-- calculation

console.log(filteredUsers()); // \['Laura', 'Linda'] <-- cached

letter.set('E'); // filteredUsers needs update on next read

console.log(filteredUsers()); // \['Emma'] <-- calculation
```

What’s happening here?

* We define two writable signals: users and letter.
* We define the computed signal filteredUsers, that depends on the signals users and letter.
* Next, we read filteredUsers the first time, triggering its calculation and returning the output specified in the code.
* Every subsequent read to filteredUsers will return its cached value.
* As we update the letter using its setter interface, we invalidate filteredUsers’ cached value.
* We read filteredUsers the last time and trigger its calculation since one of its dependencies changed.

#### Dynamic dependencies

We can dynamically specify dependencies for a signal by using conditional statements.

```
const firstName = signal('Linda');
const lastName = signal('González');
const displayLastName = signal(true);

const conditionalName = computed(() => {
 if (displayLastName()) {
   return \`Full name: ${firstName() lastName()}\`;
 } else {
   return \`First name: ${firstName()}\`;
 }
});
```

When reading conditionalName, both firstName and lastName will be dependencies of conditionalName if displayLastName is true, meaning that conditionalName should be updated whenever firstName or lastName are updated. If displayLastName is set to false, only firstName will be considered as dependency for conditionalName, ignoring any updates to lastName signal.

### Effects

An Effect is a function that is executed whenever one or more internal signals change, in order to create a new effect, we use the effect function.

```typescript
const user = signal('Clark');

effect(() => {
 console.log(\`Username is ${user()}\`); // Username is Clark
});
```

By reading the user inside our effect, we are defining a dependency, that is, the effect depends on the signal user, allowing the effect to be executed when the signal emits a new value.

You cannot set or update a signal inside an effect since that could potentially result in ExpressionChangedAfterItHasBeenChecked errors, infinite circular updates, or unnecessary change detection cycles. If you need to do it, you have to enable allowSignalWrites in the effect options.

Use case for effects:

```
@Component({...})

export class AppComponent {

 name: WritableSignal<string> = signal('Clark');

 constructor() {
   effect(() => {
     window.localStorage.setItem('name', this.name());
   })
 }

 changeName(name: string) {
   this.name.set(name);
 }
}
```

In this example, we update the localStorage with the value emitted by the name signal, allowing us to keep in sync our application state and our localStorage.

#### Injection context

To successfully register an effect, we need to do it inside an [injection context](https://angular.dev/guide/di/dependency-injection-context), the easiest way of doing so is by creating our effect in the constructor as we did in our previous example, but we have other options as well:

```
@Component({...})
export class App {
 readonly name = signal('Aljadis Tavera');
 
 private persistLocalStorageEffect: EffectRef = effect(() => {
   window.localStorage.setItem('name', this.name());
 });
}
```

By assigning the effect to a class field, we have access to the EffectRef and also we give the effect a descriptive name.

Manually initializing an effect:

If you want to initialize your effect manually, you must provide a reference to the injector via its options:

```
@Component({...})
export class App {

 readonly name = signal('Aljadis Tavera');

 private persistLocalStorageEffect!: EffectRef;

 constructor(private injector: Injector) {}

 registerEffect() {
   this.persistLocalStorageEffect = effect(() => {
     window.localStorage.setItem('name', this.name());
   }, { injector: this.injector });
 }
}
```

Destroying effect:

```
@Component({...})
export class App {

 readonly name = signal('Aljadis Tavera');

 private persistLocalStorageEffect: EffectRef = effect(() => {
     window.localStorage.setItem('name', this.name());
 });

 destroyEffect() {
   this.persistLocalStorageEffect.destroy();
 }
}
```

Note: effects are destroyed automatically when used inside a component, directive or service.

### Untracked Signals

Angular also provides a way to ignore updates to a signal by using the untracked function, this function prevents the signal from being registered as a dependency.

```
const name = signal('Adan');

const age = signal(25);

effect(() => {
  window.localStorage.setItem(
   'name-age', \`${this.name()}-${untracked(this.age)}\`
  );
});
```

Note the use of the untracked function, here’s how you can understand that example:

* When the effect is executed the first time, the localStorage will be set to the key:value ‘name-age: Adan-25’
* Then, if we change the name via the setter interface, the effect will be executed again with the new value for name.
* If we change the value of age, the effect will not be executed since age is wrapped with the untracked function.

Signals specified inside untracked should not have parentheses.

- - -

I created a small [Stackblitz project](https://stackblitz.com/edit/angular-xwgpwl?file=src%2Fmain.ts) with the concepts that we studied in this guide so you can see Angular Signals in Action! Feel free to go there and take a look.

If you enjoyed the post, let me know by clapping on it and leave any feedback in the comments, thank you for reading.