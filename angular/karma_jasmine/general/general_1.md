# Guía extensa de tests unitarios en Angular (Karma + Jasmine)

> Puedes guardar este contenido directamente como `testing-angular.md`  
> Todo el contenido está en formato Markdown.

---

## Índice rápido

1. [Chuleta rápida de Jasmine / Karma](#1-chuleta-rápida-de-jasmine--karma)
2. [Servicio simple (sin dependencias)](#2-servicio-simple-sin-dependencias)
3. [Servicio con dependencias y spies](#3-servicio-con-dependencias-y-spies)
4. [Servicio con HttpClient (GET/POST/DELETE)](#4-servicio-con-httpclient-getpostdelete)
5. [Servicio con RxJS y manejo de errores](#5-servicio-con-rxjs-y-manejo-de-errores)
6. [Servicio basado en BehaviorSubject (estado compartido)](#6-servicio-basado-en-behaviorsubject-estado-compartido)
7. [Servicio asíncrono con Promises y `fakeAsync`](#7-servicio-asíncrono-con-promises-y-fakeasync)
8. [Servicio con Observables y `done`](#8-servicio-con-observables-y-done)
9. [Componente básico con @Input](#9-componente-básico-con-input)
10. [Componente con botón y método](#10-componente-con-botón-y-método)
11. [Componente con @Output (event emitter)](#11-componente-con-output-event-emitter)
12. [Componente que usa un servicio (mock en el TestBed)](#12-componente-que-usa-un-servicio-mock-en-el-testbed)
13. [Componente con Reactive Forms](#13-componente-con-reactive-forms)
14. [Componente con Template-driven Forms](#14-componente-con-template-driven-forms)
15. [Componente con validador asíncrono](#15-componente-con-validador-asíncrono)
16. [Componente con ViewChild y acceso al DOM](#16-componente-con-viewchild-y-acceso-al-dom)
17. [Componente con OnChanges y @Input](#17-componente-con-onchanges-y-input)
18. [Componente OnPush y ChangeDetectorRef](#18-componente-onpush-y-changedetectorref)
19. [Componente con Router.navigate](#19-componente-con-routernavigate)
20. [Componente con ARIA / accesibilidad](#20-componente-con-aria--accesibilidad)
21. [Componente con setTimeout / timers](#21-componente-con-settimeout--timers)
22. [Componente con contenido proyectado (ng-content)](#22-componente-con-contenido-proyectado-ng-content)
23. [Pipe simple](#23-pipe-simple)
24. [Pipe con parámetros](#24-pipe-con-parámetros)
25. [Directiva de atributo (estilo simple)](#25-directiva-de-atributo-estilo-simple)
26. [Directiva de atributo con HostListener](#26-directiva-de-atributo-con-hostlistener)
27. [Directiva estructural (*appIfRole)](#27-directiva-estructural-appifrole)
28. [Guard de ruta (AuthGuard)](#28-guard-de-ruta-authguard)
29. [Resolver de ruta](#29-resolver-de-ruta)
30. [Test de rutas con RouterTestingModule](#30-test-de-rutas-con-routertestingmodule)
31. [HTTP Interceptor (añadir Authorization)](#31-http-interceptor-añadir-authorization)
32. [Sobrescribir providers por test (`overrideProvider`)](#32-sobrescribir-providers-por-test-overrideprovider)

---

## 1. Chuleta rápida de Jasmine / Karma

```ts
describe('Nombre del grupo de tests', () => {
  beforeEach(() => {
    // se ejecuta antes de CADA test (it)
  });

  it('debería hacer algo', () => {
    const resultado = 1 + 1;
    expect(resultado).toBe(2);
  });

  it('debería llamar a una función', () => {
    const obj = { foo: () => {} };
    spyOn(obj, 'foo');

    obj.foo();

    expect(obj.foo).toHaveBeenCalled();
  });
});
```

---

## 2. Servicio simple (sin dependencias)

```ts
// math.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
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
import { TestBed } from '@angular/core/testing';
import { MathService } from './math.service';

describe('MathService', () => {
  let service: MathService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(MathService);
  });

  it('debería crearse', () => {
    expect(service).toBeTruthy();
  });

  it('debería sumar dos números', () => {
    expect(service.sum(2, 3)).toBe(5);
  });

  it('debería detectar números pares', () => {
    expect(service.isEven(4)).toBeTrue();
    expect(service.isEven(5)).toBeFalse();
  });
});
```

---

## 3. Servicio con dependencias y spies

```ts
// logger.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LoggerService {
  log(message: string) {
    console.log(message);
  }
}
```

```ts
// user.service.ts
import { Injectable } from '@angular/core';
import { LoggerService } from './logger.service';

@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private logger: LoggerService) {}

  getUserName(): string {
    const name = 'Alice';
    this.logger.log(`User name is ${name}`);
    return name;
  }
}
```

```ts
// user.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { UserService } from './user.service';
import { LoggerService } from './logger.service';

describe('UserService', () => {
  let service: UserService;
  let loggerSpy: jasmine.SpyObj<LoggerService>;

  beforeEach(() => {
    const spy = jasmine.createSpyObj('LoggerService', ['log']);

    TestBed.configureTestingModule({
      providers: [
        UserService,
        { provide: LoggerService, useValue: spy }
      ]
    });

    service = TestBed.inject(UserService);
    loggerSpy = TestBed.inject(LoggerService) as jasmine.SpyObj<LoggerService>;
  });

  it('debería devolver nombre y loguear', () => {
    const name = service.getUserName();
    expect(name).toBe('Alice');
    expect(loggerSpy.log).toHaveBeenCalledWith('User name is Alice');
  });
});
```

---

## 4. Servicio con HttpClient (GET/POST/DELETE)

```ts
// todo.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

@Injectable({ providedIn: 'root' })
export class TodoService {
  private baseUrl = '/api/todos';

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
import { TestBed } from '@angular/core/testing';
import {
  HttpClientTestingModule,
  HttpTestingController
} from '@angular/common/http/testing';
import { TodoService, Todo } from './todo.service';

describe('TodoService', () => {
  let service: TodoService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [TodoService]
    });

    service = TestBed.inject(TodoService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('debería obtener todos (GET)', () => {
    const mockTodos: Todo[] = [
      { id: 1, title: 'Test', completed: false }
    ];

    service.getTodos().subscribe(todos => {
      expect(todos).toEqual(mockTodos);
    });

    const req = httpMock.expectOne('/api/todos');
    expect(req.request.method).toBe('GET');

    req.flush(mockTodos);
  });

  it('debería crear todo (POST)', () => {
    const newTodo: Todo = { id: 2, title: 'Nuevo', completed: false };

    service.addTodo('Nuevo').subscribe(todo => {
      expect(todo).toEqual(newTodo);
    });

    const req = httpMock.expectOne('/api/todos');
    expect(req.request.method).toBe('POST');
    expect(req.request.body).toEqual({ title: 'Nuevo', completed: false });

    req.flush(newTodo);
  });

  it('debería borrar todo (DELETE)', () => {
    service.deleteTodo(1).subscribe(res => {
      expect(res).toBeUndefined();
    });

    const req = httpMock.expectOne('/api/todos/1');
    expect(req.request.method).toBe('DELETE');

    req.flush(null);
  });
});
```

---

## 5. Servicio con RxJS y manejo de errores

```ts
// user-api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { catchError, map, of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class UserApiService {
  constructor(private http: HttpClient) {}

  getUserNameOrFallback() {
    return this.http.get<{ name: string }>('/api/user').pipe(
      map(res => res.name.toUpperCase()),
      catchError(() => of('ANÓNIMO'))
    );
  }
}
```

```ts
// user-api.service.spec.ts
import { TestBed } from '@angular/core/testing';
import {
  HttpClientTestingModule,
  HttpTestingController
} from '@angular/common/http/testing';
import { UserApiService } from './user-api.service';

describe('UserApiService', () => {
  let service: UserApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserApiService]
    });

    service = TestBed.inject(UserApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('debería devolver el nombre en mayúsculas', () => {
    service.getUserNameOrFallback().subscribe(name => {
      expect(name).toBe('ALICE');
    });

    const req = httpMock.expectOne('/api/user');
    req.flush({ name: 'Alice' });
  });

  it('debería devolver ANÓNIMO si hay error', () => {
    service.getUserNameOrFallback().subscribe(name => {
      expect(name).toBe('ANÓNIMO');
    });

    const req = httpMock.expectOne('/api/user');
    req.flush('error', { status: 500, statusText: 'Server Error' });
  });
});
```

---

## 6. Servicio basado en BehaviorSubject (estado compartido)

```ts
// state.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
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
import { TestBed } from '@angular/core/testing';
import { StateService } from './state.service';

describe('StateService', () => {
  let service: StateService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [StateService]
    });
    service = TestBed.inject(StateService);
  });

  it('debería iniciar en 0', (done: DoneFn) => {
    service.count$.subscribe(value => {
      expect(value).toBe(0);
      done();
    });
  });

  it('debería incrementar', (done: DoneFn) => {
    const resultados: number[] = [];
    const sub = service.count$.subscribe(value => resultados.push(value));

    service.increment();
    service.increment();

    expect(resultados).toEqual([0, 1, 2]);
    sub.unsubscribe();
    done();
  });

  it('debería hacer reset', () => {
    service.increment();
    service.reset();
    service.count$.subscribe(value => {
      expect(value).toBe(0);
    });
  });
});
```

---

## 7. Servicio asíncrono con Promises y `fakeAsync`

```ts
// async.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AsyncService {
  getValueAsync(): Promise<number> {
    return new Promise(resolve => setTimeout(() => resolve(42), 1000));
  }
}
```

```ts
// async.service.spec.ts
import { TestBed, fakeAsync, tick } from '@angular/core/testing';
import { AsyncService } from './async.service';

describe('AsyncService', () => {
  let service: AsyncService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [AsyncService]
    });
    service = TestBed.inject(AsyncService);
  });

  it('debería resolver 42', fakeAsync(() => {
    let result: number | undefined;

    service.getValueAsync().then(value => (result = value));

    expect(result).toBeUndefined();

    tick(1000);

    expect(result).toBe(42);
  }));
});
```

---

## 8. Servicio con Observables y `done`

```ts
// observable.service.ts
import { Injectable } from '@angular/core';
import { of } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ObservableService {
  getValue$() {
    return of(1, 2, 3);
  }
}
```

```ts
// observable.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { ObservableService } from './observable.service';

describe('ObservableService', () => {
  let service: ObservableService;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [ObservableService]
    });
    service = TestBed.inject(ObservableService);
  });

  it('debería emitir 1,2,3', (done: DoneFn) => {
    const values: number[] = [];
    service.getValue$().subscribe({
      next: v => values.push(v),
      complete: () => {
        expect(values).toEqual([1, 2, 3]);
        done();
      }
    });
  });
});
```

---

## 9. Componente básico con @Input

```ts
// hello.component.ts
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-hello',
  template: `<h1>Hello {{ name }}!</h1>`
})
export class HelloComponent {
  @Input() name = 'World';
}
```

```ts
// hello.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { HelloComponent } from './hello.component';

describe('HelloComponent', () => {
  let component: HelloComponent;
  let fixture: ComponentFixture<HelloComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HelloComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(HelloComponent);
    component = fixture.componentInstance;
  });

  it('debería mostrar World por defecto', () => {
    fixture.detectChanges();
    const h1: HTMLElement = fixture.nativeElement.querySelector('h1');
    expect(h1.textContent).toContain('Hello World!');
  });

  it('debería mostrar el nombre pasado por input', () => {
    component.name = 'Angular';
    fixture.detectChanges();
    const h1: HTMLElement = fixture.nativeElement.querySelector('h1');
    expect(h1.textContent).toContain('Hello Angular!');
  });
});
```

---

## 10. Componente con botón y método

```ts
// counter.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-counter',
  template: `
    <p data-testid="value">{{ count }}</p>
    <button (click)="increment()">Incrementar</button>
  `
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
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { CounterComponent } from './counter.component';
import { By } from '@angular/platform-browser';

describe('CounterComponent', () => {
  let component: CounterComponent;
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [CounterComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería iniciar a 0', () => {
    const value = fixture.nativeElement.querySelector('[data-testid="value"]');
    expect(value.textContent.trim()).toBe('0');
  });

  it('debería incrementar al hacer click', () => {
    const button = fixture.debugElement.query(By.css('button'));
    button.triggerEventHandler('click', null);
    fixture.detectChanges();

    const value = fixture.nativeElement.querySelector('[data-testid="value"]');
    expect(value.textContent.trim()).toBe('1');
  });
});
```

---

## 11. Componente con @Output (event emitter)

```ts
// child.component.ts
import { Component, EventEmitter, Output } from '@angular/core';

@Component({
  selector: 'app-child',
  template: `<button (click)="notify()">Notify parent</button>`
})
export class ChildComponent {
  @Output() clicked = new EventEmitter<string>();

  notify() {
    this.clicked.emit('hola padre');
  }
}
```

```ts
// child.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ChildComponent } from './child.component';
import { By } from '@angular/platform-browser';

describe('ChildComponent', () => {
  let component: ChildComponent;
  let fixture: ComponentFixture<ChildComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ChildComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(ChildComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería emitir evento al hacer click', () => {
    spyOn(component.clicked, 'emit');

    const button = fixture.debugElement.query(By.css('button'));
    button.triggerEventHandler('click', null);

    expect(component.clicked.emit).toHaveBeenCalledWith('hola padre');
  });
});
```

---

## 12. Componente que usa un servicio (mock en el TestBed)

```ts
// todo-list.component.ts
import { Component, OnInit } from '@angular/core';
import { TodoService, Todo } from '../../services/todo.service';

@Component({
  selector: 'app-todo-list',
  template: `
    <ul>
      <li *ngFor="let todo of todos">{{ todo.title }}</li>
    </ul>
  `
})
export class TodoListComponent implements OnInit {
  todos: Todo[] = [];

  constructor(private todoService: TodoService) {}

  ngOnInit(): void {
    this.todoService.getTodos().subscribe(t => (this.todos = t));
  }
}
```

```ts
// todo-list.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { of } from 'rxjs';
import { TodoListComponent } from './todo-list.component';
import { TodoService } from '../../services/todo.service';
import { By } from '@angular/platform-browser';

describe('TodoListComponent', () => {
  let component: TodoListComponent;
  let fixture: ComponentFixture<TodoListComponent>;
  let todoServiceMock: jasmine.SpyObj<TodoService>;

  beforeEach(async () => {
    const spy = jasmine.createSpyObj('TodoService', ['getTodos']);

    await TestBed.configureTestingModule({
      declarations: [TodoListComponent],
      providers: [{ provide: TodoService, useValue: spy }]
    }).compileComponents();

    fixture = TestBed.createComponent(TodoListComponent);
    component = fixture.componentInstance;
    todoServiceMock = TestBed.inject(TodoService) as jasmine.SpyObj<TodoService>;
  });

  it('debería mostrar lista de todos', () => {
    todoServiceMock.getTodos.and.returnValue(
      of([{ id: 1, title: 'Test todo', completed: false }])
    );

    fixture.detectChanges();

    const items = fixture.debugElement.queryAll(By.css('li'));
    expect(items.length).toBe(1);
    expect(items[0].nativeElement.textContent).toContain('Test todo');
  });
});
```

---

## 13. Componente con Reactive Forms

```ts
// login-form.component.ts
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

@Component({
  selector: 'app-login-form',
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email" placeholder="Email" />
      <input formControlName="password" type="password" placeholder="Password" />
      <button type="submit" [disabled]="form.invalid">Login</button>
    </form>
  `
})
export class LoginFormComponent {
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      email: ['', [Validators.required, Validators.email]],
      password: ['', Validators.required]
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
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ReactiveFormsModule } from '@angular/forms';
import { LoginFormComponent } from './login-form.component';

describe('LoginFormComponent', () => {
  let component: LoginFormComponent;
  let fixture: ComponentFixture<LoginFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [LoginFormComponent],
      imports: [ReactiveFormsModule]
    }).compileComponents();

    fixture = TestBed.createComponent(LoginFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería ser inválido por defecto', () => {
    expect(component.form.valid).toBeFalse();
  });

  it('debería ser válido con datos correctos', () => {
    component.form.setValue({
      email: 'test@example.com',
      password: '123456'
    });

    expect(component.form.valid).toBeTrue();
  });

  it('debería marcar error de email inválido', () => {
    component.form.setValue({
      email: 'no-email',
      password: '123456'
    });

    expect(component.form.valid).toBeFalse();
    expect(component.form.get('email')?.hasError('email')).toBeTrue();
  });
});
```

---

## 14. Componente con Template-driven Forms

```ts
// contact-form.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-contact-form',
  template: `
    <form #f="ngForm">
      <input name="name" ngModel required />
      <input name="email" ngModel="test@test.com" required email />
      <button [disabled]="f.invalid">Enviar</button>
    </form>
  `
})
export class ContactFormComponent {}
```

```ts
// contact-form.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FormsModule } from '@angular/forms';
import { ContactFormComponent } from './contact-form.component';
import { By } from '@angular/platform-browser';

describe('ContactFormComponent', () => {
  let component: ContactFormComponent;
  let fixture: ComponentFixture<ContactFormComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ContactFormComponent],
      imports: [FormsModule]
    }).compileComponents();

    fixture = TestBed.createComponent(ContactFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería deshabilitar botón si form es inválido', () => {
    const button = fixture.debugElement.query(By.css('button')).nativeElement;
    expect(button.disabled).toBeTrue();
  });
});
```

---

## 15. Componente con validador asíncrono

```ts
// username-async.validator.ts
import { Injectable } from '@angular/core';
import { AbstractControl, AsyncValidator, ValidationErrors } from '@angular/forms';
import { Observable, of } from 'rxjs';
import { delay, map } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UsernameAsyncValidator implements AsyncValidator {
  validate(control: AbstractControl): Observable<ValidationErrors | null> {
    const forbidden = control.value === 'admin';
    return of(forbidden).pipe(
      delay(500),
      map(isForbidden => (isForbidden ? { usernameTaken: true } : null))
    );
  }
}
```

```ts
// username-async.validator.spec.ts
import { TestBed, fakeAsync, tick } from '@angular/core/testing';
import { FormControl } from '@angular/forms';
import { UsernameAsyncValidator } from './username-async.validator';

describe('UsernameAsyncValidator', () => {
  let validator: UsernameAsyncValidator;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [UsernameAsyncValidator]
    });
    validator = TestBed.inject(UsernameAsyncValidator);
  });

  it('debería marcar error si username es admin', fakeAsync(() => {
    const control = new FormControl('admin');
    let errors: any;

    validator.validate(control).subscribe(e => (errors = e));

    tick(500);

    expect(errors).toEqual({ usernameTaken: true });
  }));

  it('debería ser válido para otros usernames', fakeAsync(() => {
    const control = new FormControl('user1');
    let errors: any;

    validator.validate(control).subscribe(e => (errors = e));

    tick(500);

    expect(errors).toBeNull();
  }));
});
```

---

## 16. Componente con ViewChild y acceso al DOM

```ts
// focus-input.component.ts
import { Component, ElementRef, ViewChild } from '@angular/core';

@Component({
  selector: 'app-focus-input',
  template: `<input #inputEl /><button (click)="focus()">Focus</button>`
})
export class FocusInputComponent {
  @ViewChild('inputEl') inputEl!: ElementRef<HTMLInputElement>;

  focus() {
    this.inputEl.nativeElement.focus();
  }
}
```

```ts
// focus-input.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FocusInputComponent } from './focus-input.component';
import { By } from '@angular/platform-browser';

describe('FocusInputComponent', () => {
  let component: FocusInputComponent;
  let fixture: ComponentFixture<FocusInputComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [FocusInputComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(FocusInputComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería llamar a focus al hacer click', () => {
    spyOn(component, 'focus').and.callThrough();

    const button = fixture.debugElement.query(By.css('button'));
    button.triggerEventHandler('click', null);

    expect(component.focus).toHaveBeenCalled();
  });
});
```

---

## 17. Componente con OnChanges y @Input

```ts
// greeting.component.ts
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-greeting',
  template: `<p>{{ greeting }}</p>`
})
export class GreetingComponent implements OnChanges {
  @Input() name = '';
  greeting = '';

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['name']) {
      this.greeting = `Hola, ${this.name}!`;
    }
  }
}
```

```ts
// greeting.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { GreetingComponent } from './greeting.component';

describe('GreetingComponent', () => {
  let component: GreetingComponent;
  let fixture: ComponentFixture<GreetingComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [GreetingComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(GreetingComponent);
    component = fixture.componentInstance;
  });

  it('debería actualizar greeting al cambiar el input', () => {
    component.name = 'Pepe';
    component.ngOnChanges({
      name: {
        currentValue: 'Pepe',
        previousValue: '',
        firstChange: true,
        isFirstChange: () => true
      }
    });
    fixture.detectChanges();

    expect(component.greeting).toBe('Hola, Pepe!');
  });
});
```

---

## 18. Componente OnPush y ChangeDetectorRef

```ts
// onpush.component.ts
import { ChangeDetectionStrategy, ChangeDetectorRef, Component, Input } from '@angular/core';

@Component({
  selector: 'app-onpush',
  template: `<p>{{ value }}</p>`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnPushComponent {
  @Input() value = '';

  constructor(private cdr: ChangeDetectorRef) {}

  updateValue(newValue: string) {
    this.value = newValue;
    this.cdr.markForCheck();
  }
}
```

```ts
// onpush.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { OnPushComponent } from './onpush.component';

describe('OnPushComponent', () => {
  let component: OnPushComponent;
  let fixture: ComponentFixture<OnPushComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [OnPushComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(OnPushComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería actualizar value mediante updateValue', () => {
    component.updateValue('Nuevo');
    fixture.detectChanges();
    const p: HTMLElement = fixture.nativeElement.querySelector('p');
    expect(p.textContent).toBe('Nuevo');
  });
});
```

---

## 19. Componente con Router.navigate

```ts
// nav.component.ts
import { Component } from '@angular/core';
import { Router } from '@angular/router';

@Component({
  selector: 'app-nav',
  template: `<button (click)="goHome()">Home</button>`
})
export class NavComponent {
  constructor(private router: Router) {}

  goHome() {
    this.router.navigate(['/home']);
  }
}
```

```ts
// nav.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { NavComponent } from './nav.component';
import { Router } from '@angular/router';
import { By } from '@angular/platform-browser';

describe('NavComponent', () => {
  let component: NavComponent;
  let fixture: ComponentFixture<NavComponent>;
  let routerMock: any;

  beforeEach(async () => {
    routerMock = { navigate: jasmine.createSpy('navigate') };

    await TestBed.configureTestingModule({
      declarations: [NavComponent],
      providers: [{ provide: Router, useValue: routerMock }]
    }).compileComponents();

    fixture = TestBed.createComponent(NavComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería navegar a /home al hacer click', () => {
    const button = fixture.debugElement.query(By.css('button'));
    button.triggerEventHandler('click', null);

    expect(routerMock.navigate).toHaveBeenCalledWith(['/home']);
  });
});
```

---

## 20. Componente con ARIA / accesibilidad

```ts
// toggle-panel.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-toggle-panel',
  template: `
    <button
      [attr.aria-expanded]="open"
      (click)="toggle()"
      id="toggle-btn"
    >
      Toggle
    </button>
    <div
      *ngIf="open"
      role="region"
      aria-labelledby="toggle-btn"
    >
      Contenido
    </div>
  `
})
export class TogglePanelComponent {
  open = false;

  toggle() {
    this.open = !this.open;
  }
}
```

```ts
// toggle-panel.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { TogglePanelComponent } from './toggle-panel.component';
import { By } from '@angular/platform-browser';

describe('TogglePanelComponent (ARIA)', () => {
  let component: TogglePanelComponent;
  let fixture: ComponentFixture<TogglePanelComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [TogglePanelComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(TogglePanelComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería tener aria-expanded="false" inicialmente', () => {
    const btn = fixture.debugElement.query(By.css('button')).nativeElement;
    expect(btn.getAttribute('aria-expanded')).toBe('false');
  });

  it('debería cambiar aria-expanded y mostrar región al hacer toggle', () => {
    const btn = fixture.debugElement.query(By.css('button'));
    btn.triggerEventHandler('click', null);
    fixture.detectChanges();

    const btnEl = btn.nativeElement;
    expect(btnEl.getAttribute('aria-expanded')).toBe('true');

    const region = fixture.debugElement.query(By.css('[role="region"]'));
    expect(region).toBeTruthy();
    expect(region.attributes['aria-labelledby']).toBe('toggle-btn');
  });
});
```

---

## 21. Componente con setTimeout / timers

```ts
// delayed-message.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-delayed-message',
  template: `<p>{{ message }}</p>`
})
export class DelayedMessageComponent {
  message = '...';

  showMessage() {
    setTimeout(() => {
      this.message = 'Hola';
    }, 1000);
  }
}
```

```ts
// delayed-message.component.spec.ts
import { ComponentFixture, TestBed, fakeAsync, tick } from '@angular/core/testing';
import { DelayedMessageComponent } from './delayed-message.component';

describe('DelayedMessageComponent', () => {
  let component: DelayedMessageComponent;
  let fixture: ComponentFixture<DelayedMessageComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [DelayedMessageComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(DelayedMessageComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('debería mostrar mensaje después de 1s', fakeAsync(() => {
    component.showMessage();
    tick(1000);
    fixture.detectChanges();

    const p: HTMLElement = fixture.nativeElement.querySelector('p');
    expect(p.textContent).toBe('Hola');
  }));
});
```

---

## 22. Componente con contenido proyectado (ng-content)

```ts
// card.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-card',
  template: `
    <div class="card">
      <ng-content></ng-content>
    </div>
  `
})
export class CardComponent {}
```

```ts
// card.component.host.ts
import { Component } from '@angular/core';

@Component({
  template: `
    <app-card>
      <span class="inner">Contenido interno</span>
    </app-card>
  `
})
export class CardHostComponent {}
```

```ts
// card.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { CardComponent } from './card.component';
import { CardHostComponent } from './card.component.host';
import { By } from '@angular/platform-browser';

describe('CardComponent (contenido proyectado)', () => {
  let fixture: ComponentFixture<CardHostComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [CardComponent, CardHostComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(CardHostComponent);
    fixture.detectChanges();
  });

  it('debería proyectar el contenido dentro de .card', () => {
    const cardDebug = fixture.debugElement.query(By.css('.card'));
    const inner = cardDebug.query(By.css('.inner'));
    expect(inner.nativeElement.textContent).toContain('Contenido interno');
  });
});
```

---

## 23. Pipe simple

```ts
// uppercase-first.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'uppercaseFirst' })
export class UppercaseFirstPipe implements PipeTransform {
  transform(value: string | null): string | null {
    if (!value) return value;
    return value.charAt(0).toUpperCase() + value.slice(1);
  }
}
```

```ts
// uppercase-first.pipe.spec.ts
import { UppercaseFirstPipe } from './uppercase-first.pipe';

describe('UppercaseFirstPipe', () => {
  const pipe = new UppercaseFirstPipe();

  it('debería capitalizar la primera letra', () => {
    expect(pipe.transform('angular')).toBe('Angular');
  });

  it('debería devolver null o vacío correctamente', () => {
    expect(pipe.transform('')).toBe('');
    expect(pipe.transform(null)).toBeNull();
  });
});
```

---

## 24. Pipe con parámetros

```ts
// truncate.pipe.ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({ name: 'truncate' })
export class TruncatePipe implements PipeTransform {
  transform(value: string, limit = 10): string {
    if (!value || value.length <= limit) {
      return value;
    }
    return value.slice(0, limit) + '…';
  }
}
```

```ts
// truncate.pipe.spec.ts
import { TruncatePipe } from './truncate.pipe';

describe('TruncatePipe', () => {
  const pipe = new TruncatePipe();

  it('debería truncar si supera el límite', () => {
    expect(pipe.transform('12345678901', 5)).toBe('12345…');
  });

  it('no debería truncar si es más corto', () => {
    expect(pipe.transform('1234', 5)).toBe('1234');
  });
});
```

---

## 25. Directiva de atributo (estilo simple)

```ts
// highlight.directive.ts
import { Directive, ElementRef, Input } from '@angular/core';

@Directive({
  selector: '[appHighlight]'
})
export class HighlightDirective {
  @Input('appHighlight') color = 'yellow';

  constructor(private el: ElementRef) {
    this.el.nativeElement.style.backgroundColor = this.color;
  }
}
```

```ts
// highlight.directive.spec.ts
import { Component } from '@angular/core';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { HighlightDirective } from './highlight.directive';
import { By } from '@angular/platform-browser';

@Component({
  template: `
    <p appHighlight="red">Texto 1</p>
    <p>Texto 2</p>
  `
})
class TestHostComponent {}

describe('HighlightDirective', () => {
  let fixture: ComponentFixture<TestHostComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HighlightDirective, TestHostComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(TestHostComponent);
    fixture.detectChanges();
  });

  it('debería aplicar color de fondo al primer párrafo', () => {
    const pDebug = fixture.debugElement.queryAll(By.css('p'))[0];
    expect(pDebug.nativeElement.style.backgroundColor).toBe('red');
  });
});
```

---

## 26. Directiva de atributo con HostListener

```ts
// hover-highlight.directive.ts
import { Directive, ElementRef, HostListener, Input } from '@angular/core';

@Directive({
  selector: '[appHoverHighlight]'
})
export class HoverHighlightDirective {
  @Input('appHoverHighlight') color = 'yellow';

  constructor(private el: ElementRef) {}

  @HostListener('mouseenter')
  onEnter() {
    this.el.nativeElement.style.backgroundColor = this.color;
  }

  @HostListener('mouseleave')
  onLeave() {
    this.el.nativeElement.style.backgroundColor = '';
  }
}
```

```ts
// hover-highlight.directive.spec.ts
import { Component } from '@angular/core';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { HoverHighlightDirective } from './hover-highlight.directive';
import { By } from '@angular/platform-browser';

@Component({
  template: `
    <p appHoverHighlight="red">Texto 1</p>
  `
})
class TestHostComponentHover {}

describe('HoverHighlightDirective', () => {
  let fixture: ComponentFixture<TestHostComponentHover>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [HoverHighlightDirective, TestHostComponentHover]
    }).compileComponents();

    fixture = TestBed.createComponent(TestHostComponentHover);
    fixture.detectChanges();
  });

  it('debería resaltar al hacer mouseenter y quitar al mouseleave', () => {
    const pDebug = fixture.debugElement.query(By.css('p'));

    pDebug.triggerEventHandler('mouseenter', {});
    fixture.detectChanges();
    expect(pDebug.nativeElement.style.backgroundColor).toBe('red');

    pDebug.triggerEventHandler('mouseleave', {});
    fixture.detectChanges();
    expect(pDebug.nativeElement.style.backgroundColor).toBe('');
  });
});
```

---

## 27. Directiva estructural (*appIfRole)

```ts
// if-role.directive.ts
import {
  Directive,
  Input,
  TemplateRef,
  ViewContainerRef
} from '@angular/core';

@Directive({
  selector: '[appIfRole]'
})
export class IfRoleDirective {
  private hasView = false;

  @Input() set appIfRole(role: string) {
    const currentUserRole = 'admin'; // ejemplo fijo
    if (role === currentUserRole && !this.hasView) {
      this.viewContainer.createEmbeddedView(this.templateRef);
      this.hasView = true;
    } else if (role !== currentUserRole && this.hasView) {
      this.viewContainer.clear();
      this.hasView = false;
    }
  }

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef
  ) {}
}
```

```ts
// if-role.directive.spec.ts
import { Component } from '@angular/core';
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { IfRoleDirective } from './if-role.directive';
import { By } from '@angular/platform-browser';

@Component({
  template: `
    <p *appIfRole="'admin'" class="admin">Admin</p>
    <p *appIfRole="'user'" class="user">User</p>
  `
})
class TestIfRoleHostComponent {}

describe('IfRoleDirective', () => {
  let fixture: ComponentFixture<TestIfRoleHostComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [IfRoleDirective, TestIfRoleHostComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(TestIfRoleHostComponent);
    fixture.detectChanges();
  });

  it('debería mostrar elemento para rol admin', () => {
    const adminElement = fixture.debugElement.query(By.css('.admin'));
    expect(adminElement).toBeTruthy();
  });

  it('no debería mostrar elemento para rol user', () => {
    const userElement = fixture.debugElement.query(By.css('.user'));
    expect(userElement).toBeNull();
  });
});
```

---

## 28. Guard de ruta (AuthGuard)

```ts
// auth.guard.ts
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { AuthService } from './auth.service';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}

  canActivate(): boolean {
    if (!this.auth.isLoggedIn()) {
      this.router.navigate(['/login']);
      return false;
    }
    return true;
  }
}
```

```ts
// auth.guard.spec.ts
import { TestBed } from '@angular/core/testing';
import { Router } from '@angular/router';
import { AuthGuard } from './auth.guard';
import { AuthService } from './auth.service';

describe('AuthGuard', () => {
  let guard: AuthGuard;
  let authServiceMock: any;
  let routerMock: any;

  beforeEach(() => {
    authServiceMock = { isLoggedIn: jasmine.createSpy() };
    routerMock = { navigate: jasmine.createSpy('navigate') };

    TestBed.configureTestingModule({
      providers: [
        AuthGuard,
        { provide: AuthService, useValue: authServiceMock },
        { provide: Router, useValue: routerMock }
      ]
    });

    guard = TestBed.inject(AuthGuard);
  });

  it('debería permitir acceso si está logueado', () => {
    authServiceMock.isLoggedIn.and.returnValue(true);

    const result = guard.canActivate();
    expect(result).toBeTrue();
    expect(routerMock.navigate).not.toHaveBeenCalled();
  });

  it('debería redirigir a /login si no está logueado', () => {
    authServiceMock.isLoggedIn.and.returnValue(false);

    const result = guard.canActivate();
    expect(result).toBeFalse();
    expect(routerMock.navigate).toHaveBeenCalledWith(['/login']);
  });
});
```

---

## 29. Resolver de ruta

```ts
// todos-resolver.service.ts
import { Injectable } from '@angular/core';
import { Resolve } from '@angular/router';
import { Observable } from 'rxjs';
import { Todo, TodoService } from '../services/todo.service';

@Injectable({ providedIn: 'root' })
export class TodosResolver implements Resolve<Todo[]> {
  constructor(private todoService: TodoService) {}

  resolve(): Observable<Todo[]> {
    return this.todoService.getTodos();
  }
}
```

```ts
// todos-resolver.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { of } from 'rxjs';
import { TodosResolver } from './todos-resolver.service';
import { TodoService } from '../services/todo.service';

describe('TodosResolver', () => {
  let resolver: TodosResolver;
  let todoServiceMock: jasmine.SpyObj<TodoService>;

  beforeEach(() => {
    const spy = jasmine.createSpyObj('TodoService', ['getTodos']);

    TestBed.configureTestingModule({
      providers: [
        TodosResolver,
        { provide: TodoService, useValue: spy }
      ]
    });

    resolver = TestBed.inject(TodosResolver);
    todoServiceMock = TestBed.inject(TodoService) as jasmine.SpyObj<TodoService>;
  });

  it('debería resolver la lista de todos', () => {
    const mockTodos = [{ id: 1, title: 'Test', completed: false }];

    todoServiceMock.getTodos.and.returnValue(of(mockTodos));

    resolver.resolve().subscribe(todos => {
      expect(todos).toEqual(mockTodos);
    });

    expect(todoServiceMock.getTodos).toHaveBeenCalled();
  });
});
```

---

## 30. Test de rutas con RouterTestingModule

```ts
// app-routing.module.ts (simplificado)
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { HomeComponent } from './home.component';
import { TodoListComponent } from './todo-list.component';

export const routes: Routes = [
  { path: 'home', component: HomeComponent },
  { path: 'todos', component: TodoListComponent },
  { path: '', redirectTo: '/home', pathMatch: 'full' }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

```ts
// app-routing.module.spec.ts
import { TestBed, fakeAsync, tick } from '@angular/core/testing';
import { RouterTestingModule } from '@angular/router/testing';
import { Router } from '@angular/router';
import { Location } from '@angular/common';
import { routes } from './app-routing.module';
import { HomeComponent } from './home.component';
import { TodoListComponent } from './todo-list.component';

describe('Rutas de la app', () => {
  let router: Router;
  let location: Location;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [RouterTestingModule.withRoutes(routes)],
      declarations: [HomeComponent, TodoListComponent]
    }).compileComponents();

    router = TestBed.inject(Router);
    location = TestBed.inject(Location);

    router.initialNavigation();
  });

  it('"" debería redirigir a /home', fakeAsync(() => {
    router.navigate(['']);
    tick();
    expect(location.path()).toBe('/home');
  }));

  it('/todos debería navegar a /todos', fakeAsync(() => {
    router.navigate(['/todos']);
    tick();
    expect(location.path()).toBe('/todos');
  }));
});
```

---

## 31. HTTP Interceptor (añadir Authorization)

```ts
// auth.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpEvent,
  HttpHandler,
  HttpInterceptor,
  HttpRequest
} from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const token = 'fake-token';
    const cloned = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
    return next.handle(cloned);
  }
}
```

```ts
// auth.interceptor.spec.ts
import { TestBed } from '@angular/core/testing';
import {
  HTTP_INTERCEPTORS,
  HttpClient
} from '@angular/common/http';
import {
  HttpClientTestingModule,
  HttpTestingController
} from '@angular/common/http/testing';
import { AuthInterceptor } from './auth.interceptor';

describe('AuthInterceptor', () => {
  let http: HttpClient;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [
        {
          provide: HTTP_INTERCEPTORS,
          useClass: AuthInterceptor,
          multi: true
        }
      ]
    });

    http = TestBed.inject(HttpClient);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('debería añadir Authorization header', () => {
    http.get('/test').subscribe();

    const req = httpMock.expectOne('/test');
    expect(req.request.headers.has('Authorization')).toBeTrue();
    expect(req.request.headers.get('Authorization')).toBe('Bearer fake-token');

    req.flush({});
  });
});
```

---

## 32. Sobrescribir providers por test (`overrideProvider`)

Ejemplo típico: cambiar el comportamiento de un servicio SOLO en un test concreto.

```ts
// time.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class TimeService {
  now(): Date {
    return new Date();
  }
}
```

```ts
// show-time.component.ts
import { Component } from '@angular/core';
import { TimeService } from './time.service';

@Component({
  selector: 'app-show-time',
  template: `<p>{{ time | date:'HH:mm' }}</p>`
})
export class ShowTimeComponent {
  time: Date;

  constructor(private timeService: TimeService) {
    this.time = this.timeService.now();
  }
}
```

```ts
// show-time.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ShowTimeComponent } from './show-time.component';
import { TimeService } from './time.service';

describe('ShowTimeComponent con overrideProvider', () => {
  let fixture: ComponentFixture<ShowTimeComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ShowTimeComponent],
      providers: [TimeService]
    }).compileComponents();
  });

  it('debería mostrar hora fija usando overrideProvider', () => {
    const fixedDate = new Date(2020, 0, 1, 10, 30);

    TestBed.overrideProvider(TimeService, {
      useValue: { now: () => fixedDate }
    });

    fixture = TestBed.createComponent(ShowTimeComponent);
    fixture.detectChanges();

    const p: HTMLElement = fixture.nativeElement.querySelector('p');
    // No comprobamos formato exacto para no depender de locale.
    expect(p.textContent).toContain('10');
  });
});
```

---

### Cómo usar este .md en tu día a día

- Si te piden **tests de servicios** → busca secciones 2, 3, 4, 5, 6, 7, 8.
- Si te piden **tests de componentes** → 9 a 22 (inputs, outputs, forms, router, ARIA, etc.).
- Si te piden **tests de directivas/pipes** → 23 a 27.
- Si te piden **tests de rutas/guards/interceptors** → 28 a 31.
- Si te piden **casos más avanzados** (override, estados, OnPush) → 6, 18, 32.
