# Guía de tests unitarios en Angular con **Jest + Angular Testing Library**

> Guarda este archivo como `testing-angular-jest-rtl.md`  
> Aquí usamos:
>
> - Jest como framework de test
> - [`@testing-library/angular`](https://testing-library.com/docs/angular-testing-library/intro/)  
>   para testear componentes de forma más parecida al uso real por parte del usuario.

---

## 1. Guía rápida de Jest + Angular Testing Library

```ts
import { render, screen, fireEvent } from "@testing-library/angular";

describe("Mi componente", () => {
  it("debería renderizar el texto", async () => {
    await render(HelloComponent, {
      componentProperties: { name: "Angular" },
    });

    expect(screen.getByText(/Hello Angular/i)).toBeInTheDocument();
  });

  it("debería reaccionar a un click", async () => {
    await render(CounterComponent);
    fireEvent.click(screen.getByRole("button", { name: /incrementar/i }));

    expect(screen.getByTestId("value").textContent).toBe("1");
  });
});
```

- Para **componentes** → `render`, `screen`, `fireEvent`, queries por rol/texto/label.
- Para **servicios, guards, interceptors** → puedes seguir usando `TestBed` como en la guía de Jest sin ATL.
- Matchers adicionales de DOM (como `toBeInTheDocument`) vienen de `@testing-library/jest-dom`.

---

## 2. Servicio simple (TestBed + Jest)

```ts
// math.service.ts
import { Injectable } from "@angular/core";

@Injectable({ providedIn: "root" })
export class MathService {
  sum(a: number, b: number): number {
    return a + b;
  }
}
```

```ts
// math.service.spec.ts
import { TestBed } from "@angular/core/testing";
import { MathService } from "./math.service";

describe("MathService (Jest + ATL proyecto)", () => {
  let service: MathService;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(MathService);
  });

  it("debería sumar", () => {
    expect(service.sum(2, 3)).toBe(5);
  });
});
```

---

## 3. Componente básico con @Input (usando `render`)

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
import { render, screen } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { HelloComponent } from "./hello.component";

describe("HelloComponent (ATL)", () => {
  it("debería mostrar World por defecto", async () => {
    await render(HelloComponent);

    expect(screen.getByText("Hello World!")).toBeInTheDocument();
  });

  it("debería mostrar el nombre pasado por input", async () => {
    await render(HelloComponent, {
      componentProperties: { name: "Angular" },
    });

    expect(screen.getByText("Hello Angular!")).toBeInTheDocument();
  });
});
```

---

## 4. Componente con botón y estado (counter)

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

  increment() {
    this.count++;
  }
}
```

```ts
// counter.component.spec.ts
import { render, screen, fireEvent } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { CounterComponent } from "./counter.component";

describe("CounterComponent (ATL)", () => {
  it("debería iniciar en 0", async () => {
    await render(CounterComponent);

    expect(screen.getByTestId("value")).toHaveTextContent("0");
  });

  it("debería incrementar al hacer click", async () => {
    await render(CounterComponent);

    fireEvent.click(screen.getByRole("button", { name: /incrementar/i }));

    expect(screen.getByTestId("value")).toHaveTextContent("1");
  });
});
```

---

## 5. Componente con @Output y comunicación padre-hijo

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
import { render, screen, fireEvent } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { ChildComponent } from "./child.component";

describe("ChildComponent (ATL)", () => {
  it("debería emitir evento al hacer click", async () => {
    const clickedHandler = jest.fn();

    await render(ChildComponent, {
      componentOutputs: {
        clicked: clickedHandler,
      },
    });

    fireEvent.click(screen.getByRole("button", { name: /notify parent/i }));

    expect(clickedHandler).toHaveBeenCalledWith("hola padre");
  });
});
```

---

## 6. Componente que usa un servicio (mock a través de providers)

```ts
// todo.service.ts
import { Injectable } from "@angular/core";
import { Observable, of } from "rxjs";

export interface Todo {
  id: number;
  title: string;
  completed: boolean;
}

@Injectable({ providedIn: "root" })
export class TodoService {
  getTodos(): Observable<Todo[]> {
    return of([]);
  }
}
```

```ts
// todo-list.component.ts
import { Component, OnInit } from "@angular/core";
import { TodoService, Todo } from "../services/todo.service";

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
import { render, screen } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { of } from "rxjs";
import { TodoListComponent } from "./todo-list.component";
import { TodoService } from "../services/todo.service";

describe("TodoListComponent (ATL)", () => {
  it("debería mostrar lista de todos", async () => {
    const todoServiceMock: Partial<TodoService> = {
      getTodos: () => of([{ id: 1, title: "Test todo", completed: false }]),
    };

    await render(TodoListComponent, {
      providers: [{ provide: TodoService, useValue: todoServiceMock }],
    });

    expect(screen.getByText("Test todo")).toBeInTheDocument();
  });
});
```

---

## 7. Formularios reactivos (Reactive Forms) con ATL

```ts
// login-form.component.ts
import { Component } from "@angular/core";
import { FormBuilder, FormGroup, Validators } from "@angular/forms";

@Component({
  selector: "app-login-form",
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <label>
        Email
        <input formControlName="email" />
      </label>
      <label>
        Password
        <input formControlName="password" type="password" />
      </label>
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
import { render, screen, fireEvent } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { ReactiveFormsModule } from "@angular/forms";
import { LoginFormComponent } from "./login-form.component";

describe("LoginFormComponent (ATL)", () => {
  it("debería deshabilitar el botón cuando es inválido", async () => {
    await render(LoginFormComponent, {
      imports: [ReactiveFormsModule],
    });

    const button = screen.getByRole("button", { name: /login/i });
    expect(button).toBeDisabled();
  });

  it("debería habilitar el botón cuando el formulario es válido", async () => {
    await render(LoginFormComponent, {
      imports: [ReactiveFormsModule],
    });

    const emailInput = screen.getByLabelText(/email/i) as HTMLInputElement;
    const passwordInput = screen.getByLabelText(
      /password/i
    ) as HTMLInputElement;

    fireEvent.input(emailInput, { target: { value: "test@example.com" } });
    fireEvent.input(passwordInput, { target: { value: "123456" } });

    const button = screen.getByRole("button", { name: /login/i });
    expect(button).toBeEnabled();
  });
});
```

---

## 8. Componente accesible con ARIA (usando queries por rol)

```ts
// toggle-panel.component.ts
import { Component } from "@angular/core";

@Component({
  selector: "app-toggle-panel",
  template: `
    <button [attr.aria-expanded]="open" (click)="toggle()" id="toggle-btn">
      Toggle
    </button>
    <div *ngIf="open" role="region" aria-labelledby="toggle-btn">Contenido</div>
  `,
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
import { render, screen, fireEvent } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { TogglePanelComponent } from "./toggle-panel.component";

describe("TogglePanelComponent (ATL + ARIA)", () => {
  it('debería tener aria-expanded="false" inicialmente', async () => {
    await render(TogglePanelComponent);

    const button = screen.getByRole("button", { name: /toggle/i });
    expect(button).toHaveAttribute("aria-expanded", "false");
    expect(screen.queryByRole("region")).toBeNull();
  });

  it("debería mostrar la región al hacer toggle", async () => {
    await render(TogglePanelComponent);

    const button = screen.getByRole("button", { name: /toggle/i });
    fireEvent.click(button);

    expect(button).toHaveAttribute("aria-expanded", "true");
    expect(screen.getByRole("region")).toBeInTheDocument();
  });
});
```

---

## 9. Componente con timers / setTimeout (usando Jest timers)

```ts
// delayed-message.component.ts
import { Component } from "@angular/core";

@Component({
  selector: "app-delayed-message",
  template: `<p>{{ message }}</p>`,
})
export class DelayedMessageComponent {
  message = "...";

  showMessage() {
    setTimeout(() => {
      this.message = "Hola";
    }, 1000);
  }
}
```

```ts
// delayed-message.component.spec.ts
import { render, screen } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { DelayedMessageComponent } from "./delayed-message.component";

describe("DelayedMessageComponent (ATL + Jest timers)", () => {
  beforeEach(() => {
    jest.useFakeTimers();
  });

  afterEach(() => {
    jest.useRealTimers();
  });

  it("debería mostrar mensaje después de 1s", async () => {
    const { fixture } = await render(DelayedMessageComponent);

    fixture.componentInstance.showMessage();

    jest.advanceTimersByTime(1000);
    fixture.detectChanges();

    expect(screen.getByText("Hola")).toBeInTheDocument();
  });
});
```

---

## 10. Directiva de atributo con HostListener (test host + ATL)

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
import { render, screen, fireEvent } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { HoverHighlightDirective } from "./hover-highlight.directive";

@Component({
  template: `<p appHoverHighlight="red">Texto 1</p>`,
})
class TestHostComponent {}

describe("HoverHighlightDirective (ATL)", () => {
  it("debería aplicar y quitar el color al hacer hover", async () => {
    await render(TestHostComponent, {
      declarations: [HoverHighlightDirective],
    });

    const p = screen.getByText("Texto 1");

    fireEvent.mouseEnter(p);
    expect(p).toHaveStyle({ backgroundColor: "red" });

    fireEvent.mouseLeave(p);
    expect(p).not.toHaveStyle({ backgroundColor: "red" });
  });
});
```

---

## 11. Pipe simple (igual que sin ATL)

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

describe("UppercaseFirstPipe (Jest + ATL project)", () => {
  const pipe = new UppercaseFirstPipe();

  it("debería capitalizar la primera letra", () => {
    expect(pipe.transform("angular")).toBe("Angular");
  });
});
```

---

## 12. Navegación con Router (usando RouterTestingModule + ATL)

```ts
// nav.component.ts
import { Component } from "@angular/core";
import { Router } from "@angular/router";

@Component({
  selector: "app-nav",
  template: `<button (click)="goHome()">Home</button>`,
})
export class NavComponent {
  constructor(private router: Router) {}

  goHome() {
    this.router.navigate(["/home"]);
  }
}
```

```ts
// nav.component.spec.ts
import { render, screen, fireEvent } from "@testing-library/angular";
import "@testing-library/jest-dom";
import { Router } from "@angular/router";
import { NavComponent } from "./nav.component";

describe("NavComponent (ATL)", () => {
  it("debería llamar a router.navigate", async () => {
    const routerMock = { navigate: jest.fn() };

    await render(NavComponent, {
      providers: [{ provide: Router, useValue: routerMock }],
    });

    fireEvent.click(screen.getByRole("button", { name: /home/i }));

    expect(routerMock.navigate).toHaveBeenCalledWith(["/home"]);
  });
});
```

---

## 13. Guard, Resolver, Interceptor (sin ATL pero dentro del mismo proyecto)

ATL está orientado a componentes. Para guards/resolvers/interceptors usas `TestBed` igual que en el archivo de Jest “puro”. Ejemplo rápido de guard:

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

describe("AuthGuard (Jest + ATL proyecto)", () => {
  let guard: AuthGuard;
  let authServiceMock: any;
  let routerMock: any;

  beforeEach(() => {
    authServiceMock = { isLoggedIn: jest.fn() };
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

  it("debería redirigir si no está logueado", () => {
    authServiceMock.isLoggedIn.mockReturnValue(false);

    const result = guard.canActivate();
    expect(result).toBe(false);
    expect(routerMock.navigate).toHaveBeenCalledWith(["/login"]);
  });
});
```

---

### Resumen mental

- **Servicios / guards / interceptors** → `TestBed` + Jest (igual que en el otro archivo).
- **Componentes / directivas** → Angular Testing Library:
  - `render(...)` con opciones (`imports`, `providers`, `declarations`, `componentInputs`, `componentOutputs`…).
  - Queries accesibles: `getByRole`, `getByText`, `getByLabelText`, `getByTestId`.
  - Interacciones: `fireEvent` (y si quieres, `userEvent`).
  - Matchers: `toBeInTheDocument`, `toHaveAttribute`, `toHaveTextContent`, `toHaveStyle`, etc.

Este archivo complementa al de Jest "puro":

- Usa aquel como referencia para **lógica de negocio, servicios, HTTP, routing**.
- Usa este para todo lo que implique **UI y comportamiento de usuario**.
