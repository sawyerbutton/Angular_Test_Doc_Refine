# Angular 测试

## 对于基本服务的测试

### 对于没有依赖的服务

- 对于服务的测试通常是单元测试中的最小单元
- 同步单元测试
- 异步单元测试
> 对于同步的测试

> 对于Observable的测试

> 对于Promise的测试

```javascript
describe('TempService', () => {
  let service: TempService;
  beforeEach(() => { service = new TempService(); });
 // 同步的测试
  it('#getValue should return real value', () => {
    expect(service.getValue()).toBe('real value');
  });
 // Observable的测试
  it('#getObservableValue should return value from observable',
    (done: DoneFn) => {
    service.getObservableValue().subscribe(value => {
      expect(value).toBe('observable value');
      done();
    });
  });
 // Promise 的测试
  it('#getPromiseValue should return value from a promise',
    (done: DoneFn) => {
    service.getPromiseValue().then(value => {
      expect(value).toBe('promise value');
      done();
    });
  });
});
```

### 对于有依赖的服务而言

- 比如

```javascript
@Injectable()
export class MasterService {
  constructor(private valueService: ValueService) { }
  getValue() { return this.valueService.getValue(); }
}
```

> MasterService把getValue方法委托给了注入的ValueSerice, 并调用了ValueService的getValue方法
- 测试的方法不止一种

> 注入真实的服务

```javascript
describe('MasterService without Angular testing support', () => {
  let masterService: MasterService;
  // 使用 new 创建了 ValueService，然后把它传给了 MasterService 的构造函数
  it('#getValue should return real value from the real service', () => {
    masterService = new MasterService(new ValueService());
    expect(masterService.getValue()).toBe('real value');
  });
});
```

> 模拟注入一个服务

```javascript
describe('MasterService without Angular testing support', () => {
  let masterService: MasterService;
  // new一个假的服务代替真实的服务，仅仅是一个例子
  it('#getValue should return faked value from a fakeService', () => {
    masterService = new MasterService(new FakeValueService());
    expect(masterService.getValue()).toBe('faked service value');
  });
  // 创建一个对象代替包含一个getValue方法 假装是一个ValueService
  it('#getValue should return faked value from a fake object', () => {
    const fake =  { getValue: () => 'fake value' };
    masterService = new MasterService(fake as ValueService);
    expect(masterService.getValue()).toBe('fake value');
  });
});
```

> 使用jasmine创建一个间谍来代替真实的ValueService依赖注入

```javascript
describe('MasterService without Angular testing support', () => {

   it('#getValue should return stubbed value from a spy', () => {
    // create `getValue` spy on an object representing the ValueService
    const valueServiceSpy =
      jasmine.createSpyObj('ValueService', ['getValue']);
    // set the value to return when the `getValue` spy is called.
    const stubValue = 'stub value';
    valueServiceSpy.getValue.and.returnValue(stubValue);
    masterService = new MasterService(valueServiceSpy);
    expect(masterService.getValue())
      .toBe(stubValue, 'service returned stub value');
    expect(valueServiceSpy.getValue.calls.count())
      .toBe(1, 'spy method was called once');
    expect(valueServiceSpy.getValue.calls.mostRecent().returnValue)
      .toBe(stubValue);
  });
});
```

- 虽然这些标准的测试技巧在隔离的环境下对服务进行单元测试很棒
- 但是，你迟早要用 Angular 的依赖注入机制来把服务注入到应用类中去，并对此进行相应的测试而不仅仅使用jasmine的方式
- Angular的测试工具集可以帮助你探索这种注入服务的工作方式。

## Angular的testbed

- 在angular中,使用DI依赖注入来创建服务是很常见的事情
- 当某个服务依赖另一个服务时,DI就会找到或者创建那个被依赖的服务,如果那个被依赖的服务还有它自己的依赖,DI也同样会找到或者创建它们(寻找服务和创建服务的区别在于他们是否已经被合理地创建)
- 作为服务的消费方你不需要关心构造函数中的参数顺序或如何创建它们,这一切DI都会帮你解决
- 但是对于服务的测试而言,你至少需要第一层的服务依赖
- 好消息是,当你在使用testbed工具创建和提供服务的时候,你可以让angular的DI进行服务的创建并且处理构造参数的顺序,麻烦的部分不需要你来考虑

### 什么是Angular testbed

- TestBed 是 Angular 测试工具中最重要的部分
- TestBed 会动态创建一个用来模拟 @NgModule 的 Angular 测试模块
- TestBed.configureTestingModule()方法接收一个元数据对象,其具有 @NgModule中的绝大多数属性

> 测试某个服务时,要在元数据的providers属性中指定一个数组，这个数组包含将要进行测试或模拟的相关服务,就像是@NgModules()中的providers一样

```javascript
let service: ValueService;

beforeEach(() => {
  TestBed.configureTestingModule({ providers: [ValueService] });
});
```

> 通过调用TestBed.get()(参数为需要的服务类）把服务注入到一个测试中

```javascript
it('should use ValueService', () => {
  service = TestBed.get(ValueService);
  expect(service.getValue()).toBe('real value');
});
```

> 如果你更倾向于把该服务作为环境配置的一部分，就把它放在beforeEach()中

```javascript
beforeEach(() => {
  TestBed.configureTestingModule({ providers: [ValueService] });
  service = TestBed.get(ValueService);
});
```

> 如果要测试一个带有依赖项的服务，那就把模拟对象放在providers数组中

```javascript
let masterService: MasterService;
let valueServiceSpy: jasmine.SpyObj<ValueService>;

beforeEach(() => {
  const spy = jasmine.createSpyObj('ValueService', ['getValue']);

  TestBed.configureTestingModule({
    // Provide both the service-to-test and its (spy) dependency
    providers: [
      MasterService,
      { provide: ValueService, useValue: spy }
    ]
  });
  // Inject both the service-to-test and its (spy) dependency
  masterService = TestBed.get(MasterService);
  valueServiceSpy = TestBed.get(ValueService);
});
```

> 这个测试会像以前一样使用这个间谍对象

```javascript
it('#getValue should return stubbed value from a spy', () => {
  const stubValue = 'stub value';
  valueServiceSpy.getValue.and.returnValue(stubValue);

  expect(masterService.getValue())
    .toBe(stubValue, 'service returned stub value');
  expect(valueServiceSpy.getValue.calls.count())
    .toBe(1, 'spy method was called once');
  expect(valueServiceSpy.getValue.calls.mostRecent().returnValue)
    .toBe(stubValue);
});
```

### 不使用beforeEach进行测试

- 大多数的测试套件都会调用beforeEach()来为每个it()测试准备前置条件,并依赖 TestBed 来创建类和注入服务
- 通过把可复用的准备代码放进一个单独的 setup 函数来代替 beforeEach()
- 本质上和beforeEach函数是一样的,推荐使用beforeEach函数

```javascript
function setup() {
  const valueServiceSpy =
    jasmine.createSpyObj('ValueService', ['getValue']);
  const stubValue = 'stub value';
  const masterService = new MasterService(valueServiceSpy);

  valueServiceSpy.getValue.and.returnValue(stubValue);
  return { masterService, stubValue, valueServiceSpy };
}
```

- setup()函数返回一个对象文字,其中包含测试可能引用的变量，例如masterService, 这样就不用在describe()的主体中定义半全局变量
- 比如`let masterService：MasterService`

> 每个测试都会在第一行调用 setup(),这个行为在测试主体和断言之前决定

```javascript
it('#getValue should return stubbed value from a spy', () => {
  const { masterService, stubValue, valueServiceSpy } = setup();
  expect(masterService.getValue())
    .toBe(stubValue, 'service returned stub value');
  expect(valueServiceSpy.getValue.calls.count())
    .toBe(1, 'spy method was called once');
  expect(valueServiceSpy.getValue.calls.mostRecent().returnValue)
    .toBe(stubValue);
});
```

> 这里使用了解构赋值来获得所需的变量

> `const { masterService, stubValue, valueServiceSpy } = setup();`

> 同时你也可以感受到这样的方法没有遵循DRY的原则,这也是为什么推荐选择beforeEach作为测试准备方法的原因

## 测试HTTP服务

### 使用jasmine spy模拟http服务

- 对远程服务器进行HTTP调用的数据服务通常会注入并委托给Angular HttpClient服务用以进行XHR调用
- 可以使用注入的HttpClient间谍测试数据服务，就像测试任何具有依赖关系的服务一样
- 举例来说

```javascript
let httpClientSpy: { get: jasmine.Spy };
let heroService: HeroService;
 
beforeEach(() => {
  // TODO: spy on other methods too
  httpClientSpy = jasmine.createSpyObj('HttpClient', ['get']);
  heroService = new HeroService(<any> httpClientSpy);
});
 
it('should return expected heroes (HttpClient called once)', () => {
  const expectedHeroes: Hero[] =
    [{ id: 1, name: 'A' }, { id: 2, name: 'B' }];
 
  httpClientSpy.get.and.returnValue(asyncData(expectedHeroes));
 
  heroService.getHeroes().subscribe(
    heroes => expect(heroes).toEqual(expectedHeroes, 'expected heroes'),
    fail
  );
  expect(httpClientSpy.get.calls.count()).toBe(1, 'one call');
});
 
it('should return an error when the server returns a 404', () => {
  const errorResponse = new HttpErrorResponse({
    error: 'test 404 error',
    status: 404, statusText: 'Not Found'
  });
 
  httpClientSpy.get.and.returnValue(asyncError(errorResponse));
 
  heroService.getHeroes().subscribe(
    heroes => fail('expected an error, not heroes'),
    error  => expect(error.message).toContain('test 404 error')
  );
});
```

> HttpService 中的方法会返回 Observables,订阅这些方法返回的可观察对象会让它开始执行并且断言这些方法是成功还是失败(如果你不订阅一个Observable,那么这个Observable就不会被触发)
> subscribe() 方法接受一个成功回调 (next) 和一个失败 (error) 回调,要确保同时提供了这两个回调以便捕获错误,如果忽略这些异步调用中未捕获的错误，测试运行器可能会得出不同的测试结论(影响测试的质量)

### HttpClientTestingModule

- 数据服务和HttpClient之间的扩展交互可能很难用jasmine spy进行模拟
- HttpClientTestingModule 可以让这些测试场景变得更加可控
- 这一部分放在httpClient中再仔细描述

## 如何测试组件

- 组件与 Angular 应用中的其它部分不同是由 HTML 模板和 TypeScript 类组成的
- 组件其实是指模板加上与其合作的类,要想对组件进行充分的测试就要测试它们能否如预期的那样协作
- 这些测试需要在浏览器的DOM中创建组件的宿主元素（就像 Angular 所做的那样)然后检查组件类和 DOM 的交互是否如同它在模板中期待的样子
- Angular 的 TestBed 为所有这些类型的测试提供了基础设施,但是很多情况下可以单独测试组件类本身而不必涉及 DOM, 这样就已经可以用一种更加简单清晰的方式来验证该组件的大多数行为

### 如何单独测试组建类

- 测试组建类和测试服务类其实很相似
- LightswitchComponent组件中当用户点击按钮时会切换灯的开关状态

```javascript
@Component({
  selector: 'lightswitch-comp',
  template: `
    <button (click)="clicked()">Click me!</button>
    <span>{{message}}</span>`
})
export class LightswitchComponent {
  isOn = false;
  clicked() { this.isOn = !this.isOn; }
  get message() { return `The light is ${this.isOn ? 'On' : 'Off'}`; }
}
```

> 对于这个组件而言要测试 clicked() 方法能否正确切换灯的开关状态(isOn的值)
> 该组件类没有依赖,当测试一个没有依赖的服务时会用 new 来创建它调用它的API然后对它的公开状态进行断言,同样组件类也可以这么做

```javascript
describe('LightswitchComp', () => {
  it('#clicked() should toggle #isOn', () => {
    const comp = new LightswitchComponent();
    expect(comp.isOn).toBe(false, 'off at first');
    comp.clicked();
    expect(comp.isOn).toBe(true, 'on after click');
    comp.clicked();
    expect(comp.isOn).toBe(false, 'off after second click');
  });

  it('#clicked() should set #message to "is on"', () => {
    const comp = new LightswitchComponent();
    expect(comp.message).toMatch(/is off/i, 'off at first');
    comp.clicked();
    expect(comp.message).toMatch(/is on/i, 'on after clicked');
  });
});
```

### 如何测试父子组件中的子组件

- 子组件呈现在父组件的模板中,子组件中把一个属性绑定到了 @Input 属性上，并且通过 @Output 属性监听选中某属性时的事件

```javascript
export class DashboardHeroComponent {
  @Input() hero: Hero;
  @Output() selected = new EventEmitter<Hero>();
  click() { this.selected.emit(this.hero); }
}
```

- 并不一定需要完整创建它或其父组件去测试子组件
- 例如

```javascript
it('raises the selected event when clicked', () => {
  const comp = new DashboardHeroComponent();
  const hero: Hero = { id: 42, name: 'Test' };
  comp.hero = hero;

  comp.selected.subscribe(selectedHero => expect(selectedHero).toBe(hero));
  comp.click();
});
```

- 当组件有依赖时可能要使用 TestBed 来同时创建该组件及其依赖

> 下面的 WelcomeComponent 依赖于 UserService,并通过它知道要打招呼的那位用户的名字

```javascript
export class WelcomeComponent  implements OnInit {
  welcome: string;
  constructor(private userService: UserService) { }

  ngOnInit(): void {
    this.welcome = this.userService.isLoggedIn ?
      'Welcome, ' + this.userService.user.name : 'Please log in.';
  }
}
```

> 可能要先创建一个满足本组件最小需求的模拟版本UserService

```javascript
class MockUserService {
  isLoggedIn = true;
  user = { name: 'Test User'};
};
```

> 然后在 TestBed 的配置中提供并注入该组件和该服务

```javascript
beforeEach(() => {
  TestBed.configureTestingModule({
    // provide the component-under-test and dependent service
    providers: [
      WelcomeComponent,
      { provide: UserService, useClass: MockUserService }
    ]
  });
  // inject both the component and the dependent service.
  comp = TestBed.get(WelcomeComponent);
  userService = TestBed.get(UserService);
});
```

```javascript
it('should not have welcome message after construction', () => {
  expect(comp.welcome).toBeUndefined();
});

it('should welcome logged in user after Angular calls ngOnInit', () => {
  comp.ngOnInit();
  expect(comp.welcome).toContain(userService.user.name);
});

it('should ask user to log in if not logged in after ngOnInit', () => {
  userService.isLoggedIn = false;
  comp.ngOnInit();
  expect(comp.welcome).not.toContain(userService.user.name);
  expect(comp.welcome).toContain('log in');
});
```

### 组件 DOM 的测试

- 测试组件类看起来就像测试服务那样简单,但那是在不考虑整个DOM的状况下
- 组件还要和 DOM 以及其它组件进行交互,只涉及类的测试可以告诉你组件类的行为是否正常但是不能告诉你组件是否能正常渲染,响应用户的输入和查询或与它的父组件和子组件是否正常交互
- 如果只涉及ts类的测试，那么你无法判断

> Lightswitch.clicked() 是否真的绑定到了某些用户可以接触到的东西
> Lightswitch.message 是否真的显示出来了
> 用户真的可以选择 DashboardHeroComponent 中显示的某个英雄吗
> 英雄的名字是否如预期般显示出来了（比如是否大写）
> WelcomeComponent 的模板是否显示了欢迎信息

- 这些问题对于上面这种简单的组件来说当然没有问题,不过很多组件和它们模板中所描述的 DOM 元素之间会有复杂的交互,当组件的状态发生变化时,会导致一些 HTML 出现和消失
- 因此不得不创建那些与组件相关的 DOM 元素,必须检查 DOM 来确认组件的状态能在恰当的时机正常显示出来,并且必须通过屏幕来仿真用户的交互以判断这些交互是否如预期的那般工作

> 写这类测试需要用到 TestBed 的附加功能以及其它测试助手

- angular中一个默认生成的组件测试文件可能很大,但是拆分来看其实分为基础部分和高级部分
- 比如

```javascript
import { async, ComponentFixture, TestBed } from '@angular/core/testing';
import { BannerComponent } from './banner.component';

describe('BannerComponent', () => {
  let component: BannerComponent;
  let fixture: ComponentFixture<BannerComponent>;

  beforeEach(async(() => {
    TestBed.configureTestingModule({
      declarations: [ BannerComponent ]
    })
    .compileComponents();
  }));

  beforeEach(() => {
    fixture = TestBed.createComponent(BannerComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeDefined();
  });
});
```

> 上面的测试文件中之后最后三行才是真正用来测试组件的
> 用来断言angular可以创造这个组件
> 现在可以将这个测试文件缩减成

```typescript
describe('BannerComponent (minimal)', () => {
  it('should create', () => {
    TestBed.configureTestingModule({
      declarations: [ BannerComponent ]
    });
    const fixture = TestBed.createComponent(BannerComponent);
    const component = fixture.componentInstance;
    expect(component).toBeDefined();
  });
});
```

> 传给TestBed.configureTestingModule 的元数据对象中只声明了 BannerComponent这个待测试的组件

```javascript
TestBed.configureTestingModule({
  declarations: [ BannerComponent ]
});
```

> 默认不用声明或导入任何其它的东西,默认的测试模块中已经预先配置好了一些东西，比如来自 @angular/platform-browser 的 BrowserModule

> 之后将会调用带有导入模块、服务提供商和更多可声明对象的 TestBed.configureTestingModule() 来满足测试的需要 还可以用可选的 override 方法对这些配置进行微调

- 在配置好 TestBed 之后可以调用它的 createComponent() 方法
- TestBed.createComponent() 会创建一个 BannerComponent 的实例并把相应的元素添加到测试运行器的 DOM 中，然后返回一个 ComponentFixture 对象

```javascript
const fixture = TestBed.createComponent(BannerComponent);
```

- 请注意
- `在调用了 createComponent 之后就不能再重新配置`
- `TestBedcreateComponent 方法冻结了当前的 TestBed 定义,关闭它才能再进行后续配置`
- `不能再调用任何 TestBed 的后续配置方法了,不能调 configureTestingModule()不能调 get(),也不能调用任何 override... 方法,如果试图这么做TestBed 就会抛出错误`

> ComponentFixture 是一个测试挽具（就像马车缰绳,用来与所创建的组件及其 DOM 元素进行交互

> 可以通过测试夹具(fixture)来访问该组件的实例,并用 Jasmine 的 expect 语句来确保其存在

```javascript
const component = fixture.componentInstance;
expect(component).toBeDefined();
```

- 随着该组件的成长将会添加更多测试,除了为每个测试都复制一份 TestBed 测试之外，还可以把它们重构成 Jasmine 的 beforeEach() 中的prepare语法以及一些支持性变量

```javascript
describe('BannerComponent (with beforeEach)', () => {
  let component: BannerComponent;
  let fixture: ComponentFixture<BannerComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [ BannerComponent ]
    });
    fixture = TestBed.createComponent(BannerComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeDefined();
  });
});
```

> 比如,添加一个测试,用它从 fixture.nativeElement 中获取组件的元素并查找是否存在所预期的文本内容

```javascript
it('should contain "banner works!"', () => {
  const bannerElement: HTMLElement = fixture.nativeElement;
  expect(bannerElement.textContent).toContain('banner works!');
});
```

> nativeElement
- ComponentFixture.nativeElement 的值是 any 类型的

- Angular 在编译期间没办法知道 nativeElement 是哪种 HTML 元素,甚至是否 是HTML 元素(比如可能是 SVG 元素)也未可知

- 当前的例子都是为运行在浏览器中而设计的,因此 nativeElement 的值一定会是 HTMLElement 及其派生类

- 如果确定它是某种 HTMLElement就可以用标准的 querySelector 在元素树中进行探索和处理了

- 下面这个测试就会调用 HTMLElement.querySelector 来获取 <p> 元素，并在其中查找 Banner 文本

```javascript
it('should have <p> with "banner works!"', () => {
  const bannerElement: HTMLElement = fixture.nativeElement;
  const p = bannerElement.querySelector('p');
  expect(p.textContent).toEqual('banner works!');
});
```