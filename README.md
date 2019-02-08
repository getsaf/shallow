# shallow-render

![logo](logo.png)

Angular 5+ testing made easy with shallow rendering and easy mocking.

[![Build Status](https://travis-ci.org/getsaf/shallow-render.svg?branch=master)](https://travis-ci.org/getsaf/shallow-render)
[![npm version](https://badge.fury.io/js/shallow-render.svg)](https://www.npmjs.com/package/shallow-render)

---
## The Docs
* [API Docs](https://github.com/getsaf/shallow-render/wiki)
* [StackBlitz Playground](https://stackblitz.com/github/getsaf/shallow-render-stackblitz)
* [Release Notes](https://github.com/getsaf/shallow-render/wiki/Releases)

## The problem
Testing in Angular is **HARD**. TestBed is powerful but its use in component specs ends with lots of duplication.

Here's a standard TestBed spec for a component that uses a few other components, a directive and a pipe and handles click events:
```typescript
describe('MyComponent', () => {
  beforeEach(async => {
    return TestBed.configureTestModule({
      imports: [
        SomeModuleWithDependencies,
      ],
      declarations: [
        TestHostComponent,
        MyComponent, // <-- All I want to do is test this!!
        // We either must list all our dependencies here
        // -- OR --
        // Use NO_ERRORS_SCHEMA which allows any HTML to be used
        // even if it is invalid!
        ButtonComponent,
        LinkComponent,
        FooDirective,
        BarPipe,
      ],
      providers: [
        MyService
      ]
    }).compileComponents().then(() => {
      let myService = TestBed.get(MyService); // Not type safe
      spyOn(myService, 'foo').and.returnValue('mocked foo');
    });
  });

  it('renders a link with the provided label text', () => {
    const fixture = TestBed.createComponent(TestHostComponent);
    fixture.componentInstance.labelText = 'my text';
    fixture.detectChanges();
    const link = fixture.debugElement.query(By.css('a'));

    expect(a.nativeElement.innerText).toBe('my text');
  });
  
  it('sends "foo" to bound click events', () => {
    const fixture = TestBed.createComponent(TestHostComponent);
    spyOn(fixture.componentInstance, 'handleClick');
    fixture.detectChanges();
    const myComponentElement = fixture.debugElement.query(By.directive(MyComponent));
    myComponentElement.click();
    
    expect(fixture.componentInstance.handleClick).toHaveBeenCalledWith('foo');
  });
});

@Component({
  template: `
    <my-component [linkText]="linkText" (click)="handleClick($event)">
    </my-component>
  `
})
class TestHostComponent {
  linkLabel: string;
  handleClick() {}
}
```

Whew!!! That was a lot of boilerplate. Here's just some of the issues:
* Our TestBed module looks very similar if not identical to the `NgModule` I've probably already added `MyComponent` to.  Total module duplication.
* Since I've duplicated my module in my spec, I'm not actually sure the real module was setup correctly.
* I've used REAL components and services in my spec which means I have not isolated the component I'm interested in testing.
	* This also means I have to follow, and provide all the dependencies of those real components to the `TestBed` module.
* I had to create a `TestHostComponent` so I could pass bindings into my actual component.
* My `TestBed` boilerplate code-length exceeded my actual test code-length.

## The Solution
We should mock everything we can except for the component in test and that should be **EASY**. Our modules already define the environment in which our components live. They should be *reused*, not *rebuilt* in our specs.

Here's the same specs using `shallow-render`:
```typescript
describe('MyComponent', () => {
  let shallow: Shallow<MyComponent>;
  
  beforeEach(() => {
    shallow = new Shallow(MyComponent, MyModule);
  });

  it('renders a link with the provided label text', async () => {
    const {find} = await shallow.render(`<my-component linkText="my text"></my-component>`);

    expect(find('a').nativeElement.innerText).toBe('my text');
  });

  it('sends "foo" to bound click events', async () => {
    const {element, bindings} = await shallow.render(
      `<my-component (click)="handleClick($event)"></my-component>`,
      { bind: { handleClick: () => {} } }
    );
    element.click();

    expect(bindings.handleClick).toHaveBeenCalledWith('foo');
  });
});
```

Here's the difference:
* Reuses (and verifies) `MyModule` contains your component and all its dependencies.
* All components inside `MyModule` are mocked using the awesome [ng-mocks](https://github.com/ike18t/ng-mocks) library. This is what makes the rendering "shallow".
* The tests have much less boilerplate which makes the specs easier to follow.
* The HTML used to render the component is IN THE SPEC and easy to find.
	* This means specs now double examples of how to use your component.
