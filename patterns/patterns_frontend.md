# Padrões de Frontend

## Fluxo Completo de um Módulo

Para cada módulo novo, sempre criar nesta ordem:

1. Model/Interface (`core/models`)
2. Service (`features/nome/services`)
3. NgRx Actions (`features/nome/store`)
4. NgRx Reducer (`features/nome/store`)
5. NgRx Selectors (`features/nome/store`)
6. NgRx Effects (`features/nome/store`)
7. Components (`features/nome/components`)
8. Routes (`features/nome/nome.routes.ts`)
9. Registrar no app.routes.ts (lazy load)

---

## Model

```typescript
// core/models/client.model.ts
export interface Client {
  id: string;
  name: string;
  phone: string;
  email?: string;
  birthDate?: string;
  notes?: string;
  active: boolean;
  createdAt: string;
}

export interface CreateClientRequest {
  name: string;
  phone: string;
  email?: string;
  birthDate?: string;
  notes?: string;
}
```

---

## Service

```typescript
// features/clients/services/client.service.ts
@Injectable({ providedIn: 'root' })
export class ClientService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = '/api/clients';

  list(search?: string, page = 0, size = 20): Observable<Page<Client>> {
    const params = new HttpParams()
      .set('page', page)
      .set('size', size)
      .set('search', search ?? '');
    return this.http.get<Page<Client>>(this.baseUrl, { params });
  }

  findById(id: string): Observable<Client> {
    return this.http.get<Client>(`${this.baseUrl}/${id}`);
  }

  create(request: CreateClientRequest): Observable<Client> {
    return this.http.post<Client>(this.baseUrl, request);
  }

  update(id: string, request: CreateClientRequest): Observable<Client> {
    return this.http.put<Client>(`${this.baseUrl}/${id}`, request);
  }

  delete(id: string): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${id}`);
  }
}
```

---

## NgRx Actions

```typescript
// features/clients/store/client.actions.ts
export const loadClients = createAction(
  '[Clients] Load Clients',
  props<{ search?: string; page?: number }>()
);
export const loadClientsSuccess = createAction(
  '[Clients] Load Clients Success',
  props<{ clients: Client[]; total: number }>()
);
export const loadClientsFailure = createAction(
  '[Clients] Load Clients Failure',
  props<{ error: string }>()
);

export const createClient = createAction(
  '[Clients] Create Client',
  props<{ request: CreateClientRequest }>()
);
export const createClientSuccess = createAction(
  '[Clients] Create Client Success',
  props<{ client: Client }>()
);
export const createClientFailure = createAction(
  '[Clients] Create Client Failure',
  props<{ error: string }>()
);
```

---

## NgRx Reducer

```typescript
// features/clients/store/client.reducer.ts
export interface ClientState {
  clients: Client[];
  total: number;
  loading: boolean;
  error: string | null;
  selectedClient: Client | null;
}

const initialState: ClientState = {
  clients: [],
  total: 0,
  loading: false,
  error: null,
  selectedClient: null,
};

export const clientReducer = createReducer(
  initialState,
  on(loadClients, state => ({ ...state, loading: true, error: null })),
  on(loadClientsSuccess, (state, { clients, total }) => ({
    ...state, loading: false, clients, total
  })),
  on(loadClientsFailure, (state, { error }) => ({
    ...state, loading: false, error
  }))
);
```

---

## NgRx Effects

```typescript
// features/clients/store/client.effects.ts
@Injectable()
export class ClientEffects {
  private actions$ = inject(Actions);
  private clientService = inject(ClientService);

  loadClients$ = createEffect(() =>
    this.actions$.pipe(
      ofType(loadClients),
      switchMap(({ search, page }) =>
        this.clientService.list(search, page).pipe(
          map(response => loadClientsSuccess({
            clients: response.content,
            total: response.totalElements
          })),
          catchError(error => of(loadClientsFailure({ error: error.message })))
        )
      )
    )
  );
}
```

---

## Component

```typescript
// features/clients/components/client-list/client-list.component.ts
@Component({
  selector: 'app-client-list',
  standalone: true,
  imports: [CommonModule, MatTableModule, MatButtonModule, ReactiveFormsModule],
  templateUrl: './client-list.component.html',
})
export class ClientListComponent implements OnInit {
  private store = inject(Store);

  clients$ = this.store.select(selectAllClients);
  loading$ = this.store.select(selectClientsLoading);
  searchControl = new FormControl('');

  ngOnInit() {
    this.store.dispatch(loadClients({}));

    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      distinctUntilChanged()
    ).subscribe(search => {
      this.store.dispatch(loadClients({ search: search ?? undefined }));
    });
  }
}
```

---

## Roteamento Lazy Load

```typescript
// features/clients/client.routes.ts
export const clientRoutes: Routes = [
  { path: '', component: ClientListComponent },
  { path: 'novo', component: ClientFormComponent },
  { path: ':id/editar', component: ClientFormComponent },
];

// app.routes.ts
{
  path: 'clientes',
  loadChildren: () => import('./features/clients/client.routes').then(m => m.clientRoutes),
  canActivate: [authGuard]
}
```

---

## Pipes Customizados

```typescript
// Moeda brasileira
{{ valor | currencyBr }}  // → R$ 1.250,00

// Data em PT-BR
{{ data | datePtbr }}     // → 21/05/2026
```
