# Angular Patterns — ABP Framework (NextX)

Angular 13 + ABP 5.2.2 + ng-zorro-antd patterns for the NextX project.

---

## Table of Contents

- [ABP Module Pattern](#abp-module-pattern)
- [Lazy-Loaded Feature Modules](#lazy-loaded-feature-modules)
- [ABP Proxy Services](#abp-proxy-services)
- [ng-zorro-antd Components](#ng-zorro-antd-components)
- [NgRx State Management](#ngrx-state-management)
- [ABP Permissions & Authorization](#abp-permissions--authorization)
- [ABP Localization](#abp-localization)
- [DevExtreme Data Grids](#devextreme-data-grids)
- [Apollo GraphQL](#apollo-graphql)
- [Shared Module Conventions](#shared-module-conventions)
- [Forms: Reactive Patterns](#forms-reactive-patterns)

---

## ABP Module Pattern

Every feature follows this structure:

```
pages/feature-name/
├── feature-name.module.ts         # NgModule, lazy loaded
├── feature-name-routing.module.ts # Routes
├── feature-name.component.ts
├── feature-name.component.html
└── components/                    # Sub-components
```

### Feature Module Boilerplate

```typescript
// pages/products/products.module.ts
import { NgModule } from '@angular/core';
import { SharedModule } from '../../shared/shared.module';
import { ProductsRoutingModule } from './products-routing.module';
import { ProductsComponent } from './products.component';

@NgModule({
  declarations: [ProductsComponent],
  imports: [SharedModule, ProductsRoutingModule],
})
export class ProductsModule {}
```

### Routing (Lazy Load)

```typescript
// app-routing.module.ts
{
  path: 'products',
  loadChildren: () =>
    import('./pages/products/products.module').then(m => m.ProductsModule),
  canActivate: [AuthGuard],
}
```

---

## Lazy-Loaded Feature Modules

Register the route with ABP's route provider to appear in the navigation menu:

```typescript
// route.provider.ts
import { RoutesService, eLayoutType } from '@abp/ng.core';

export function configureRoutes(routes: RoutesService) {
  routes.add([
    {
      path: '/products',
      name: '::Menu:Products',
      iconClass: 'fas fa-box',
      layout: eLayoutType.application,
      requiredPolicy: 'NextX.Products',
    },
  ]);
}
```

---

## ABP Proxy Services

Generated from the backend API. Regenerate after any API changes:

```bash
abp generate-proxy -t ng
```

Proxies land in `src/app/proxy/`. Use them directly — never call HttpClient manually for ABP APIs:

```typescript
import { ProductService } from '../proxy/products/product.service';
import { CreateUpdateProductDto, ProductDto } from '../proxy/products/models';

@Component({ ... })
export class ProductsComponent implements OnInit {
  products: ProductDto[] = [];

  constructor(private productService: ProductService) {}

  ngOnInit() {
    this.productService.getList({ maxResultCount: 100 }).subscribe(result => {
      this.products = result.items;
    });
  }

  create(dto: CreateUpdateProductDto) {
    this.productService.create(dto).subscribe(() => this.ngOnInit());
  }
}
```

---

## ng-zorro-antd Components

Primary UI library. Always import the specific NZ module needed:

### Data Table

```typescript
// Import in feature module
import { NzTableModule } from 'ng-zorro-antd/table';
import { NzButtonModule } from 'ng-zorro-antd/button';

// Template
<nz-table
  #table
  [nzData]="products"
  [nzLoading]="loading"
  [nzPageSize]="10"
  nzShowPagination
>
  <thead>
    <tr>
      <th>Name</th>
      <th>Price</th>
      <th>Actions</th>
    </tr>
  </thead>
  <tbody>
    <tr *ngFor="let item of table.data">
      <td>{{ item.name }}</td>
      <td>{{ item.price | currency }}</td>
      <td>
        <button nz-button nzType="primary" (click)="edit(item)">Edit</button>
        <button nz-button nzDanger (click)="delete(item.id)">Delete</button>
      </td>
    </tr>
  </tbody>
</nz-table>
```

### Modal (Create/Edit)

```typescript
import { NzModalService } from 'ng-zorro-antd/modal';

constructor(private modal: NzModalService) {}

openCreateModal() {
  const ref = this.modal.create({
    nzTitle: 'Create Product',
    nzContent: ProductFormComponent,
    nzFooter: null,
    nzWidth: 720,
  });

  ref.afterClose.subscribe(result => {
    if (result) this.loadData();
  });
}
```

### Form with Validation Feedback

```html
<form nz-form [formGroup]="form" (ngSubmit)="submit()">
  <nz-form-item>
    <nz-form-label [nzSm]="6" nzRequired>Name</nz-form-label>
    <nz-form-control [nzSm]="14" nzErrorTip="Name is required">
      <input nz-input formControlName="name" />
    </nz-form-control>
  </nz-form-item>
  <button nz-button nzType="primary" [disabled]="form.invalid">Save</button>
</form>
```

### Notification

```typescript
import { NzNotificationService } from 'ng-zorro-antd/notification';

constructor(private notification: NzNotificationService) {}

showSuccess() {
  this.notification.success('Success', 'Product saved.');
}
```

---

## NgRx State Management

The project uses NgRx for shared UI state (grid selections, filters).

### Pattern: Grid State

```typescript
// core/state/grid.actions.ts
import { createAction, props } from '@ngrx/store';

export const setSelectedRows = createAction(
  '[Grid] Set Selected Rows',
  props<{ ids: string[] }>()
);

// core/state/grid.reducer.ts
import { createReducer, on } from '@ngrx/store';

export interface GridState { selectedIds: string[]; }
const initialState: GridState = { selectedIds: [] };

export const gridReducer = createReducer(
  initialState,
  on(setSelectedRows, (state, { ids }) => ({ ...state, selectedIds: ids }))
);

// Usage in component
import { Store } from '@ngrx/store';
import { setSelectedRows } from '../../core/state/grid.actions';

constructor(private store: Store) {}

onSelectionChange(ids: string[]) {
  this.store.dispatch(setSelectedRows({ ids }));
}
```

---

## ABP Permissions & Authorization

### Define Permission (Backend — Application.Contracts)

```csharp
// NextXPermissions.cs
public static class NextXPermissions
{
    public const string GroupName = "NextX";
    public static class Products
    {
        public const string Default = GroupName + ".Products";
        public const string Create = Default + ".Create";
        public const string Edit   = Default + ".Edit";
        public const string Delete = Default + ".Delete";
    }
}
```

### Guard in Angular Route

```typescript
import { PermissionGuard } from '@abp/ng.core';

{
  path: 'products',
  loadChildren: () => import('./pages/products/products.module').then(m => m.ProductsModule),
  canActivate: [PermissionGuard],
  data: { requiredPolicy: 'NextX.Products' },
}
```

### Hide UI Elements

```html
<!-- Template -->
<button *abpPermission="'NextX.Products.Create'" nz-button (click)="openCreate()">
  Add Product
</button>
```

### Check Programmatically

```typescript
import { ConfigStateService } from '@abp/ng.core';

constructor(private config: ConfigStateService) {}

canCreate(): boolean {
  return this.config.getGrantedPolicy('NextX.Products.Create');
}
```

---

## ABP Localization

### Add Localization Key (Backend)

Add to `Domain.Shared/Localization/NextX/en.json`:

```json
{
  "Culture": "en",
  "Texts": {
    "Menu:Products": "Products",
    "NewProduct": "New Product",
    "ProductName": "Product Name"
  }
}
```

### Use in Templates

```html
{{ '::Menu:Products' | abpLocalization }}
{{ '::ProductName' | abpLocalization }}
```

### Use in Components

```typescript
import { LocalizationService } from '@abp/ng.core';

constructor(private l10n: LocalizationService) {}

getLabel(): string {
  return this.l10n.instant('::ProductName');
}
```

---

## DevExtreme Data Grids

Used for complex grids with grouping, export, and inline editing.

### Basic DxDataGrid

```html
<dx-data-grid
  [dataSource]="products"
  [showBorders]="true"
  [rowAlternationEnabled]="true"
  [allowColumnReordering]="true"
  (onRowUpdated)="onRowUpdated($event)"
>
  <dxo-editing mode="row" [allowUpdating]="true" [allowDeleting]="true" />
  <dxo-export [enabled]="true" fileName="products" />
  <dxo-search-panel [visible]="true" />
  <dxo-paging [pageSize]="20" />

  <dxi-column dataField="name" caption="Name" />
  <dxi-column dataField="price" caption="Price" dataType="number" format="currency" />
  <dxi-column dataField="createdAt" caption="Created" dataType="date" />
</dx-data-grid>
```

### Server-Side Paging with CustomStore

```typescript
import CustomStore from 'devextreme/data/custom_store';

dataSource = new CustomStore({
  key: 'id',
  load: (opts) => this.productService.getList({
    skipCount: opts.skip,
    maxResultCount: opts.take,
    filter: opts.searchValue,
  }).toPromise().then(r => ({ data: r.items, totalCount: r.totalCount })),
  remove: (key) => this.productService.delete(key).toPromise(),
});
```

---

## Apollo GraphQL

Used for complex queries that span multiple entities.

### Query Example

```typescript
import { Apollo, gql } from 'apollo-angular';

const PRODUCTS_QUERY = gql`
  query GetProducts($filter: String) {
    products(filter: $filter) {
      id
      name
      price
      category { name }
    }
  }
`;

@Component({ ... })
export class ProductsComponent {
  products$ = this.apollo.watchQuery({
    query: PRODUCTS_QUERY,
    variables: { filter: '' },
  }).valueChanges.pipe(map(r => r.data['products']));

  constructor(private apollo: Apollo) {}
}
```

### Mutation

```typescript
const CREATE_PRODUCT = gql`
  mutation CreateProduct($input: CreateProductInput!) {
    createProduct(input: $input) { id name }
  }
`;

create(input: any) {
  this.apollo.mutate({ mutation: CREATE_PRODUCT, variables: { input } })
    .subscribe(() => this.refresh());
}
```

---

## Shared Module Conventions

`src/app/shared/shared.module.ts` re-exports common modules to avoid repetition:

```typescript
// What to always import from SharedModule (already re-exported):
// - CommonModule
// - FormsModule, ReactiveFormsModule
// - AbpModule, ThemeSharedModule
// - NzIconModule, NzSpinModule, NzButtonModule

// Feature modules import only SharedModule + their specific NZ modules
@NgModule({
  imports: [
    SharedModule,
    NzTableModule,
    NzModalModule,
    NzFormModule,
  ],
  declarations: [ProductsComponent],
})
export class ProductsModule {}
```

---

## Forms: Reactive Patterns

### Typed Reactive Form

```typescript
import { FormBuilder, FormGroup, Validators } from '@angular/forms';

interface ProductForm {
  name: string;
  price: number;
  categoryId: string;
}

@Component({ ... })
export class ProductFormComponent {
  form: FormGroup;

  constructor(private fb: FormBuilder) {
    this.form = this.fb.group({
      name: ['', [Validators.required, Validators.maxLength(128)]],
      price: [0, [Validators.required, Validators.min(0)]],
      categoryId: [null, Validators.required],
    });
  }

  submit() {
    if (this.form.invalid) return;
    const value = this.form.value as ProductForm;
    // call proxy service
  }
}
```

### Patch Form for Edit

```typescript
ngOnInit() {
  if (this.editId) {
    this.productService.get(this.editId).subscribe(p => {
      this.form.patchValue({
        name: p.name,
        price: p.price,
        categoryId: p.categoryId,
      });
    });
  }
}
```
