# Guía de tests unitarios en Angular con **Jest** (sin Angular Testing Library)

> Puedes guardar este archivo como `testing-angular-jest.md`  
> Todo el contenido está preparado para un proyecto Angular que usa **Jest** en lugar de Karma/Jasmine  
> (por ejemplo con `jest-preset-angular`).

---

## 1. Guía rápida de Jest

```ts
describe("grupo de tests", () => {
  beforeEach(() => {
    // se ejecuta antes de cada test
  });

  test("debería hacer algo", () => {
    const resultado = 1 + 1;
    expect(resultado).toBe(2);
  });

  it("debería llamar a una función", () => {
    const obj = { foo: () => {} };
    const spy = jest.spyOn(obj, "foo");

    obj.foo();

    expect(spy).toHaveBeenCalled();
  });
});
```

Diferencias principales respecto a Jasmine:

- Espías: `jest.fn()`, `jest.spyOn()`
- Matchers muy parecidos: `toBe`, `toEqual`, `toHaveBeenCalledWith`, etc.
- Timers: `jest.useFakeTimers()`, `jest.advanceTimersByTime(ms)` (además de `fakeAsync/tick` de Angular).
- Mocks: `jest.mock('modulo')`, `jest.fn()` como implementación falsa.

---

## 2. Servicio simple (sin dependencias)

```ts
// math.service.ts
import { Injectable } from "@angular/core";

@Injectable({ providedIn: "root" })
export class MathService {
  sum(a: number, b: number): number {
    return a + b;
  }

  isEven(n: number): boolean {
    return n % 2 === 0;
  }
}
```

```ts
// math.service.spec.ts
import { TestBed } from "@angular/core/testing";
import { MathService } from "./math.service";

describe("MathService (Jest)", () => {
  let service: MathService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(MathService);
  });

  it("debería crearse", () => {
    expect(service).toBeTruthy();
  });

  it("debería sumar dos números", () => {
    expect(service.sum(2, 3)).toBe(5);
  });

  it("debería detectar números pares", () => {
    expect(service.isEven(4)).toBe(true);
    expect(service.isEven(5)).toBe(false);
  });
});
```

---

## 3. Servicio con dependencias y espías de Jest

```ts
// logger.service.ts
import { Injectable } from "@angular/core";

@Injectable({ providedIn: "root" })
export class LoggerService {
  log(message: string) {
    console.log(message);
  }
}
```

```ts
// user.service.ts
import { Injectable } from "@angular/core";
import { LoggerService } from "./logger.service";

@Injectable({ providedIn: "root" })
export class UserService {
  constructor(private logger: LoggerService) {}

  getUserName(): string {
    const name = "Alice";
    this.logger.log(`User name is ${name}`);
    return name;
  }
}
```

```ts
// user.service.spec.ts
import { TestBed } from "@angular/core/testing";
import { UserService } from "./user.service";
import { LoggerService } from "./logger.service";

describe("UserService (Jest)", () => {
  let service: UserService;
  let loggerMock: jest.Mocked<LoggerService>;

  beforeEach(() => {
    const logger: jest.Mocked<LoggerService> = {
      log: jest.fn(),
    };

    TestBed.configureTestingModule({
      providers: [UserService, { provide: LoggerService, useValue: logger }],
    });

    service = TestBed.inject(UserService);
    loggerMock = TestBed.inject(LoggerService) as jest.Mocked<LoggerService>;
  });

  it("debería devolver nombre y loguear", () => {
    const name = service.getUserName();

    expect(name).toBe("Alice");
    expect(loggerMock.log).toHaveBeenCalledWith("User name is Alice");
  });
});
```

---

## 4. Servicio con HttpClient (GET/POST/DELETE)

```ts
// todo.service.ts
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { Observable } from "rxjs";

export interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

@Injectable({ providedIn: "root" })
export class TodoService {
  private baseUrl = "/api/todos";

  constructor(private http: HttpClient) {}

  getTodos(): Observable<Todo[]> {
    return this.http.get<Todo[]>(this.baseUrl);
  }

  addTodo(title: string): Observable<Todo> {
    return this.http.post<Todo>(this.baseUrl, { title, completed: false });
  }

  deleteTodo(id: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

```ts
// todo.service.spec.ts
import { TestBed } from "@angular/core/testing";
import {
  HttpClientTestingModule,
  HttpTestingController,
} from "@angular/common/http/testing";
import { TodoService, Todo } from "./todo.service";

describe("TodoService (Jest)", () => {
  let service: TodoService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [TodoService],
    });

    service = TestBed.inject(TodoService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it("debería obtener todos (GET)", () => {
    const mockTodos: Todo[] = [{ id: 1, title: "Test", completed: false }];

    service.getTodos().subscribe((todos) => {
      expect(todos).toEqual(mockTodos);
    });

    const req = httpMock.expectOne("/api/todos");
    expect(req.request.method).toBe("GET");

    req.flush(mockTodos);
  });

  it("debería crear todo (POST)", () => {
    const newTodo: Todo = { id: 2, title: "Nuevo", completed: false };

    service.addTodo("Nuevo").subscribe((todo) => {
      expect(todo).toEqual(newTodo);
    });

    const req = httpMock.expectOne("/api/todos");
    expect(req.request.method).toBe("POST");
    expect(req.request.body).toEqual({ title: "Nuevo", completed: false });

    req.flush(newTodo);
  });

  it("debería borrar todo (DELETE)", () => {
    service.deleteTodo(1).subscribe((res) => {
      expect(res).toBeUndefined();
    });

    const req = httpMock.expectOne("/api/todos/1");
    expect(req.request.method).toBe("DELETE");

    req.flush(null);
  });
});
```

---

## 5. Servicio con RxJS y manejo de errores

```ts
// user-api.service.ts
import { Injectable } from "@angular/core";
import { HttpClient } from "@angular/common/http";
import { catchError, map, of } from "rxjs";

@Injectable({ providedIn: "root" })
export class UserApiService {
  constructor(private http: HttpClient) {}

  getUserNameOrFallback() {
    return this.http.get<{ name: string }>("/api/user").pipe(
      map((res) => res.name.toUpperCase()),
      catchError(() => of("ANÓNIMO"))
    );
  }
}
```

```ts
// user-api.service.spec.ts
import { TestBed } from "@angular/core/testing";
import {
  HttpClientTestingModule,
  HttpTestingController,
} from "@angular/common/http/testing";
import { UserApiService } from "./user-api.service";

describe("UserApiService (Jest)", () => {
  let service: UserApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserApiService],
    });

    service = TestBed.inject(UserApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it("debería devolver el nombre en mayúsculas", () => {
    service.getUserNameOrFallback().subscribe((name) => {
      expect(name).toBe("ALICE");
    });

    const req = httpMock.expectOne("/api/user");
    req.flush({ name: "Alice" });
  });

  it("debería devolver ANÓNIMO si hay error", () => {
    service.getUserNameOrFallback().subscribe((name) => {
      expect(name).toBe("ANÓNIMO");
    });

    const req = httpMock.expectOne("/api/user");
    req.flush("error", { status: 500, statusText: "Server Error" });
  });
});
```

---

## 6. Servicio basado en BehaviorSubject (estado compartido)

```ts
// state.service.ts
import { Injectable } from "@angular/core";
import { BehaviorSubject } from "rxjs";

@Injectable({ providedIn: "root" })
export class StateService {
  private countSubject = new BehaviorSubject<number>(0);
  count$ = this.countSubject.asObservable();

  increment() {
    this.countSubject.next(this.countSubject.value + 1);
  }

  reset() {
    this.countSubject.next(0);
  }
}
```

```ts
// state.service.spec.ts
import { TestBed } from "@angular/core/testing";
import { StateService } from "./state.service";

describe("StateService (Jest)", () => {
  let service: StateService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [StateService],
    });
    service = TestBed.inject(StateService);
  });

  it("debería iniciar en 0", (done) => {
    service.count$.subscribe((value) => {
      expect(value).toBe(0);
      done();
    });
  });

  it("debería incrementar", () => {
    const resultados: number[] = [];
    const sub = service.count$.subscribe((value) => resultados.push(value));

    service.increment();
    service.increment();

    expect(resultados).toEqual([0, 1, 2]);
    sub.unsubscribe();
  });

  it("debería hacer reset", (done) => {
    service.increment();
    service.reset();
    service.count$.subscribe((value) => {
      expect(value).toBe(0);
      done();
    });
  });
});
```

---

## 7. Servicio asíncrono con Promises y timers de Jest

```ts
// async.service.ts
import { Injectable } from "@angular/core";

@Injectable({ providedIn: "root" })
export class AsyncService {
  getValueAsync(): Promise<number> {
    return new Promise((resolve) => setTimeout(() => resolve(42), 1000));
  }
}
```

```ts
// async.service.spec.ts
import { TestBed } from "@angular/core/testing";
import { AsyncService } from "./async.service";

describe("AsyncService (Jest timers)", () => {
  let service: AsyncService;

  beforeEach(() => {
    jest.useFakeTimers();
    TestBed.configureTestingModule({
      providers: [AsyncService],
    });
    service = TestBed.inject(AsyncService);
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it("debería resolver 42 usando timers de Jest", async () => {
    const promise = service.getValueAsync();

    jest.advanceTimersByTime(1000);

    await expect(promise).resolves.toBe(42);
  });
});
```

---

## 8. Servicio con Observables y `done`

```ts
// observable.service.ts
import { Injectable } from "@angular/core";
import { of } from "rxjs";

@Injectable({ providedIn: "root" })
export class ObservableService {
  getValue$() {
    return of(1, 2, 3);
  }
}
```

```ts
// observable.service.spec.ts
import { TestBed } from "@angular/core/testing";
import { ObservableService } from "./observable.service";

describe("ObservableService (Jest)", () => {
  let service: ObservableService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [ObservableService],
    });
    service = TestBed.inject(ObservableService);
  });

  it("debería emitir 1,2,3", (done) => {
    const values: number[] = [];
    service.getValue$().subscribe({
      next: (v) => values.push(v),
      complete: () => {
        expect(values).toEqual([1, 2, 3]);
        done();
      },
    });
  });
});
```

---

## 9. Componente básico con @Input

```ts
// hello.component.ts
import { Component, Input } from "@angular/core";

@Component({
  selector: "app-hello",
  template: `<h1>Hello {{ name }}!</h1>`,
})
export class HelloComponent {
  @Input() name = "World";
}
```

```ts
// hello.component.spec.ts
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { HelloComponent } from "./hello.component";

describe("HelloComponent (Jest)", () => {
  let component: HelloComponent;
  let fixture: ComponentFixture<HelloComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HelloComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(HelloComponent);
    component = fixture.componentInstance;
  });

  it("debería mostrar World por defecto", () => {
    fixture.detectChanges();
    const h1: HTMLElement = fixture.nativeElement.querySelector("h1");
    expect(h1.textContent).toContain("Hello World!");
  });

  it("debería mostrar el nombre pasado por input", () => {
    component.name = "Angular";
    fixture.detectChanges();
    const h1: HTMLElement = fixture.nativeElement.querySelector("h1");
    expect(h1.textContent).toContain("Hello Angular!");
  });
});
```

---

## 10. Componente con botón y método

```ts
// counter.component.ts
import { Component } from "@angular/core";

@Component({
  selector: "app-counter",
  template: `
    <p data-testid="value">{{ count }}</p>
    <button (click)="increment()">Incrementar</button>
  `,
})
export class CounterComponent {
  count = 0;

  increment(): void {
    this.count++;
  }
}
```

```ts
// counter.component.spec.ts
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { CounterComponent } from "./counter.component";
import { By } from "@angular/platform-browser";

describe("CounterComponent (Jest)", () => {
  let component: CounterComponent;
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [CounterComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it("debería iniciar a 0", () => {
    const value = fixture.nativeElement.querySelector('[data-testid="value"]');
    expect(value.textContent.trim()).toBe("0");
  });

  it("debería incrementar al hacer click", () => {
    const button = fixture.debugElement.query(By.css("button"));
    button.triggerEventHandler("click", null);
    fixture.detectChanges();

    const value = fixture.nativeElement.querySelector('[data-testid="value"]');
    expect(value.textContent.trim()).toBe("1");
  });
});
```

---

## 11. Componente con @Output (EventEmitter)

```ts
// child.component.ts
import { Component, EventEmitter, Output } from "@angular/core";

@Component({
  selector: "app-child",
  template: `<button (click)="notify()">Notify parent</button>`,
})
export class ChildComponent {
  @Output() clicked = new EventEmitter<string>();

  notify() {
    this.clicked.emit("hola padre");
  }
}
```

```ts
// child.component.spec.ts
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { ChildComponent } from "./child.component";
import { By } from "@angular/platform-browser";

describe("ChildComponent (Jest)", () => {
  let component: ChildComponent;
  let fixture: ComponentFixture<ChildComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ChildComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(ChildComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it("debería emitir evento al hacer click", () => {
    const emitSpy = jest.spyOn(component.clicked, "emit");

    const button = fixture.debugElement.query(By.css("button"));
    button.triggerEventHandler("click", null);

    expect(emitSpy).toHaveBeenCalledWith("hola padre");
  });
});
```

---

## 12. Componente que usa un servicio (mock en el TestBed)

```ts
// todo-list.component.ts
import { Component, OnInit } from "@angular/core";
import { TodoService, Todo } from "../../services/todo.service";

@Component({
  selector: "app-todo-list",
  template: `
    <ul>
      <li *ngFor="let todo of todos">{{ todo.title }}</li>
    </ul>
  `,
})
export class TodoListComponent implements OnInit {
  todos: Todo[] = [];

  constructor(private todoService: TodoService) {}

  ngOnInit(): void {
    this.todoService.getTodos().subscribe((t) => (this.todos = t));
  }
}
```

```ts
// todo-list.component.spec.ts
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { of } from "rxjs";
import { TodoListComponent } from "./todo-list.component";
import { TodoService } from "../../services/todo.service";
import { By } from "@angular/platform-browser";

describe("TodoListComponent (Jest)", () => {
  let component: TodoListComponent;
  let fixture: ComponentFixture<TodoListComponent>;
  let todoServiceMock: jest.Mocked<TodoService>;

  beforeEach(async () => {
    const spy: jest.Mocked<TodoService> = {
      getTodos: jest.fn(),
      addTodo: jest.fn(),
      deleteTodo: jest.fn(),
    } as any;

    await TestBed.configureTestingModule({
      declarations: [TodoListComponent],
      providers: [{ provide: TodoService, useValue: spy }],
    }).compileComponents();

    fixture = TestBed.createComponent(TodoListComponent);
    component = fixture.componentInstance;
    todoServiceMock = TestBed.inject(TodoService) as jest.Mocked<TodoService>;
  });

  it("debería mostrar lista de todos", () => {
    todoServiceMock.getTodos.mockReturnValue(
      of([{ id: 1, title: "Test todo", completed: false }]) as any
    );

    fixture.detectChanges();

    const items = fixture.debugElement.queryAll(By.css("li"));
    expect(items.length).toBe(1);
    expect(items[0].nativeElement.textContent).toContain("Test todo");
  });
});
```

---

## 13. Componente con Reactive Forms

```ts
// login-form.component.ts
import { Component } from "@angular/core";
import { FormBuilder, FormGroup, Validators } from "@angular/forms";

@Component({
  selector: "app-login-form",
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email" placeholder="Email" />
      <input
        formControlName="password"
        type="password"
        placeholder="Password"
      />
      <button type="submit" [disabled]="form.invalid">Login</button>
    </form>
  `,
})
export class LoginFormComponent {
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      email: ["", [Validators.required, Validators.email]],
      password: ["", Validators.required],
    });
  }

  submit(): void {
    if (this.form.valid) {
      console.log(this.form.value);
    }
  }
}
```

```ts
// login-form.component.spec.ts
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { ReactiveFormsModule } from "@angular/forms";
import { LoginFormComponent } from "./login-form.component";

describe("LoginFormComponent (Jest)", () => {
  let component: LoginFormComponent;
  let fixture: ComponentFixture<LoginFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [LoginFormComponent],
      imports: [ReactiveFormsModule],
    }).compileComponents();

    fixture = TestBed.createComponent(LoginFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it("debería ser inválido por defecto", () => {
    expect(component.form.valid).toBe(false);
  });

  it("debería ser válido con datos correctos", () => {
    component.form.setValue({
      email: "test@example.com",
      password: "123456",
    });

    expect(component.form.valid).toBe(true);
  });
});
```

---

## 14. Pipe simple

```ts
// uppercase-first.pipe.ts
import { Pipe, PipeTransform } from "@angular/core";

@Pipe({ name: "uppercaseFirst" })
export class UppercaseFirstPipe implements PipeTransform {
  transform(value: string | null): string | null {
    if (!value) return value;
    return value.charAt(0).toUpperCase() + value.slice(1);
  }
}
```

```ts
// uppercase-first.pipe.spec.ts
import { UppercaseFirstPipe } from "./uppercase-first.pipe";

describe("UppercaseFirstPipe (Jest)", () => {
  const pipe = new UppercaseFirstPipe();

  it("debería capitalizar la primera letra", () => {
    expect(pipe.transform("angular")).toBe("Angular");
  });

  it("debería devolver null o vacío correctamente", () => {
    expect(pipe.transform("")).toBe("");
    expect(pipe.transform(null)).toBeNull();
  });
});
```

---

## 15. Directiva de atributo con HostListener

```ts
// hover-highlight.directive.ts
import { Directive, ElementRef, HostListener, Input } from "@angular/core";

@Directive({
  selector: "[appHoverHighlight]",
})
export class HoverHighlightDirective {
  @Input("appHoverHighlight") color = "yellow";

  constructor(private el: ElementRef) {}

  @HostListener("mouseenter")
  onEnter() {
    this.el.nativeElement.style.backgroundColor = this.color;
  }

  @HostListener("mouseleave")
  onLeave() {
    this.el.nativeElement.style.backgroundColor = "";
  }
}
```

```ts
// hover-highlight.directive.spec.ts
import { Component } from "@angular/core";
import { ComponentFixture, TestBed } from "@angular/core/testing";
import { HoverHighlightDirective } from "./hover-highlight.directive";
import { By } from "@angular/platform-browser";

@Component({
  template: `<p appHoverHighlight="red">Texto 1</p>`,
})
class TestHostComponentHover {}

describe("HoverHighlightDirective (Jest)", () => {
  let fixture: ComponentFixture<TestHostComponentHover>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HoverHighlightDirective, TestHostComponentHover],
    }).compileComponents();

    fixture = TestBed.createComponent(TestHostComponentHover);
    fixture.detectChanges();
  });

  it("debería resaltar al hacer mouseenter y quitar al mouseleave", () => {
    const pDebug = fixture.debugElement.query(By.css("p"));

    pDebug.triggerEventHandler("mouseenter", {});
    fixture.detectChanges();
    expect(pDebug.nativeElement.style.backgroundColor).toBe("red");

    pDebug.triggerEventHandler("mouseleave", {});
    fixture.detectChanges();
    expect(pDebug.nativeElement.style.backgroundColor).toBe("");
  });
});
```

---

## 16. Guard de ruta (AuthGuard)

```ts
// auth.guard.ts
import { Injectable } from "@angular/core";
import { CanActivate, Router } from "@angular/router";
import { AuthService } from "./auth.service";

@Injectable({ providedIn: "root" })
export class AuthGuard implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}

  canActivate(): boolean {
    if (!this.auth.isLoggedIn()) {
      this.router.navigate(["/login"]);
      return false;
    }
    return true;
  }
}
```

```ts
// auth.guard.spec.ts
import { TestBed } from "@angular/core/testing";
import { Router } from "@angular/router";
import { AuthGuard } from "./auth.guard";
import { AuthService } from "./auth.service";

describe("AuthGuard (Jest)", () => {
  let guard: AuthGuard;
  let authServiceMock: jest.Mocked<AuthService>;
  let routerMock: any;

  beforeEach(() => {
    authServiceMock = { isLoggedIn: jest.fn() } as any;
    routerMock = { navigate: jest.fn() };

    TestBed.configureTestingModule({
      providers: [
        AuthGuard,
        { provide: AuthService, useValue: authServiceMock },
        { provide: Router, useValue: routerMock },
      ],
    });

    guard = TestBed.inject(AuthGuard);
  });

  it("debería permitir acceso si está logueado", () => {
    authServiceMock.isLoggedIn.mockReturnValue(true);

    const result = guard.canActivate();
    expect(result).toBe(true);
    expect(routerMock.navigate).not.toHaveBeenCalled();
  });

  it("debería redirigir a /login si no está logueado", () => {
    authServiceMock.isLoggedIn.mockReturnValue(false);

    const result = guard.canActivate();
    expect(result).toBe(false);
    expect(routerMock.navigate).toHaveBeenCalledWith(["/login"]);
  });
});
```

---

## 17. HTTP Interceptor (añadir Authorization)

```ts
// auth.interceptor.ts
import { Injectable } from "@angular/core";
import {
  HttpEvent,
  HttpHandler,
  HttpInterceptor,
  HttpRequest,
} from "@angular/common/http";
import { Observable } from "rxjs";

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const token = "fake-token";
    const cloned = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` },
    });
    return next.handle(cloned);
  }
}
```

```ts
// auth.interceptor.spec.ts
import { TestBed } from "@angular/core/testing";
import { HTTP_INTERCEPTORS, HttpClient } from "@angular/common/http";
import {
  HttpClientTestingModule,
  HttpTestingController,
} from "@angular/common/http/testing";
import { AuthInterceptor } from "./auth.interceptor";

describe("AuthInterceptor (Jest)", () => {
  let http: HttpClient;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        {
          provide: HTTP_INTERCEPTORS,
          useClass: AuthInterceptor,
          multi: true,
        },
      ],
    });

    http = TestBed.inject(HttpClient);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it("debería añadir Authorization header", () => {
    http.get("/test").subscribe();

    const req = httpMock.expectOne("/test");
    expect(req.request.headers.has("Authorization")).toBe(true);
    expect(req.request.headers.get("Authorization")).toBe("Bearer fake-token");

    req.flush({});
  });
});
```

---

## 18. Test de rutas con RouterTestingModule

```ts
// app-routing.module.ts (simplificado)
import { NgModule } from "@angular/core";
import { RouterModule, Routes } from "@angular/router";
import { HomeComponent } from "./home.component";
import { TodoListComponent } from "./todo-list.component";

export const routes: Routes = [
  { path: "home", component: HomeComponent },
  { path: "todos", component: TodoListComponent },
  { path: "", redirectTo: "/home", pathMatch: "full" },
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule],
})
export class AppRoutingModule {}
```

```ts
// app-routing.module.spec.ts
import { TestBed, fakeAsync, tick } from "@angular/core/testing";
import { RouterTestingModule } from "@angular/router/testing";
import { Router } from "@angular/router";
import { Location } from "@angular/common";
import { routes } from "./app-routing.module";
import { HomeComponent } from "./home.component";
import { TodoListComponent } from "./todo-list.component";

describe("Rutas de la app (Jest)", () => {
  let router: Router;
  let location: Location;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [RouterTestingModule.withRoutes(routes)],
      declarations: [HomeComponent, TodoListComponent],
    }).compileComponents();

    router = TestBed.inject(Router);
    location = TestBed.inject(Location);

    router.initialNavigation();
  });

  it('"" debería redirigir a /home', fakeAsync(() => {
    router.navigate([""]);
    tick();
    expect(location.path()).toBe("/home");
  }));

  it("/todos debería navegar a /todos", fakeAsync(() => {
    router.navigate(["/todos"]);
    tick();
    expect(location.path()).toBe("/todos");
  }));
});
```

---

### Resumen

- Usa **TestBed** igual que con Karma/Jasmine, solo cambia la **API de Jest** (`jest.fn`, `jest.spyOn`, `expect` de Jest).
- Para timers complejos, puedes elegir entre:
  - `fakeAsync`/`tick` de Angular
  - o `jest.useFakeTimers()`/`jest.advanceTimersByTime()` de Jest.
- Para mocks de servicios, usa `jest.Mocked<T>` o simples objetos con `jest.fn()`.

Este archivo te sirve como "traducción mental" de los ejemplos de Jasmine/Karma a Jest puro (sin Angular Testing Library).
