# Ngrx Effect ShoppingList

This project was generated with [Angular CLI](https://github.com/angular/angular-cli) version 9.1.15.

## Development server

Run `ng serve` for a dev server. Navigate to `http://localhost:4200/`. The app will automatically reload if you change any of the source files.

## Setting up the project
This app builds on Part 1: 'https://github.com/manish2bharti/ngrx-store-shopping-list' and adds the use of @ngrx/effects to handle side-effects within our application.

We'll be building out functionality to deal with asynchronous requests, loading indicators, and more. For example, whilst loading, we can change the colour of our drop shadow:

##What's a side effect?
Prior to adding @ngrx/effects, we firstly have to consider what a side effect is.

A lot of the time this can be boiled down to - **am I dealing with the network or another asynchronous task?**

By passing side-effects to our **@Effect** middleware and isolating this from our component(s), we're able to keep our components lighter and more responsive to change.

We're able to do this as **Effects** listens to every **Action** dispatched within our **Store**. These actions can then be filtered and we can then use this to drive state changes.

With that in mind, let's upgrade our **@ngrx/store**** Shopping List to use **@ngrx/effects**, starting with our API.

##Http and json-server
In order to persist our shopping items, we'll need a database and API of some form. As we're in prototype land, using something like **json-server** is a great choice here.

**json-server** is a file-based database that exposes a REST API based on the JSON schema. In the root of our project (/), create a **db.json** with the following content:
```
{
  "shopping": []
}
```

Then install **json-server** globally on your machine using npm or yarn:

```
npm i json-server -g
```

Finally, start your database and API by opening another terminal window and typing:

```
$ json-server db.json

\{^_^}/ hi!

Loading db.json
Done

Resources
http://localhost:3000/shopping

Home
http://localhost:3000

Type s + enter at any time to create a snapshot of the database
```

If we make a request to http://localhost:3000/shopping (using curl, Postman, or something similar), we get back an empty array:
```
$ curl http://localhost:3000/shopping

[]   
```
If I go ahead and add an item and do the same, we get:
```
{
  "shopping": [
    {
      "id": 1,
      "name": "Diet Coke"
    }
  ]
}

$ json-server db.json

$ curl http://localhost:3000/shopping

[
  {
    "id": 1,
    "name": "Diet Coke"
  }
]
```

When we change the database manually from the filesystem we have to restart it, but adding items via the API doesn't require a restart.

We can also do things like:
```
$ curl https://localhost:3000/shopping/1

{
  "id": 1,
  "name": "Diet Coke"
}
```

Now that we're confident in our prototype API, we can go ahead and create a ShoppingService with the Angular CLI to interface with this.

##ShoppingService
Inside of your terminal, run the following:
```
$ ng generate service Shopping
```

We can then use the HttpClient to interface with our API. I've added three core functions, get, post and delete:

```
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { ShoppingItem } from './store/models/shopping-item.model';
import { delay } from 'rxjs/operators';

@Injectable({
  providedIn: 'root'
})
export class ShoppingService {

  private SHOPPING_URL = "http://localhost:3000/shopping"

  constructor(private http: HttpClient) { }

  getShoppingItems() {
    return this.http.get<Array<ShoppingItem>>(this.SHOPPING_URL)
      .pipe(
      delay(500)
    )
  }

  addShoppingItem(shoppingItem: ShoppingItem) {
    return this.http.post(this.SHOPPING_URL, shoppingItem)
      .pipe(
        delay(500)
      )
  }

  deleteShoppingItem(id: string) {
    return this.http.delete(`${this.SHOPPING_URL}/${id}`)
      .pipe(
        delay(500)
      )
  }
}
```
Why is there a delay here?

I want to show an example of a loading state within our UI, although it isn't necessary, it's a fun detail. Without the delay, this'd be instantaneous and harder to see.

Finally, we'll need to import the HttpClientModule inside of our app.module.ts:

```
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  // Omitted
  imports: [
    HttpClientModule
  ]
})
```

##Async-friendly Actions
Let's change up our ShoppingActionTypes to be more async-friendly:
```
export enum ShoppingActionTypes {
  LOAD_SHOPPING = '[SHOPPING] Load Shopping',
  LOAD_SHOPPING_SUCCESS = '[SHOPPING] Load Shopping Success',
  LOAD_SHOPPING_FAILURE = '[SHOPPING] Load Shopping Failure',
  ADD_ITEM = '[SHOPPING] Add Item',
  ADD_ITEM_SUCCESS = '[SHOPPING] Add Item Success',
  ADD_ITEM_FAILURE = '[SHOPPING] Add Item Failure',
  DELETE_ITEM = '[SHOPPING] Delete Item',
  DELETE_ITEM_SUCCESS = '[SHOPPING] Delete Item Success',
  DELETE_ITEM_FAILURE = '[SHOPPING] Delete Item Failure'
}
```
As an example, this now gives us the potential to call the LOAD_SHOPPING action and based on the server response, it should either dispatch a LOAD_SHOPPING_SUCCESS or LOAD_SHOPPING_FAILURE action with a supporting payload.

Let's export these actions so that they can be dispatched:

```
export class LoadShoppingAction implements Action {
  readonly type = ShoppingActionTypes.LOAD_SHOPPING
}
export class LoadShoppingSuccessAction implements Action {
  readonly type = ShoppingActionTypes.LOAD_SHOPPING_SUCCESS

  constructor(public payload: Array<ShoppingItem>) {}

}
export class LoadShoppingFailureAction implements Action {
  readonly type = ShoppingActionTypes.LOAD_SHOPPING_FAILURE
  
  constructor(public payload: Error) {}
}

export class AddItemAction implements Action {
  readonly type = ShoppingActionTypes.ADD_ITEM

  constructor(public payload: ShoppingItem) { }
}
export class AddItemSuccessAction implements Action {
  readonly type = ShoppingActionTypes.ADD_ITEM_SUCCESS

  constructor(public payload: ShoppingItem) { }
}
export class AddItemFailureAction implements Action {
  readonly type = ShoppingActionTypes.ADD_ITEM_FAILURE

  constructor(public payload: Error) { }
}

export class DeleteItemAction implements Action {
  readonly type = ShoppingActionTypes.DELETE_ITEM

  constructor(public payload: string) { }
}

export class DeleteItemSuccessAction implements Action {
  readonly type = ShoppingActionTypes.DELETE_ITEM_SUCCESS

  constructor(public payload: string) { }
}
export class DeleteItemFailureAction implements Action {
  readonly type = ShoppingActionTypes.DELETE_ITEM_FAILURE

  constructor(public payload: string) { }
}

export type ShoppingAction = AddItemAction |
  AddItemSuccessAction |
  AddItemFailureAction |
  DeleteItemAction |
  DeleteItemSuccessAction |
  DeleteItemFailureAction |
  LoadShoppingAction |
  LoadShoppingFailureAction |
  LoadShoppingSuccessAction
  ```
  
##Updating our AppState
As we're dealing with asynchronous data, we have the option to show the fact that we're loading and a potential error inside of our UI.

When the user interacts with our list, we'll be updating the box-shadow with a new color cause we're fancy like that!

```
import { ShoppingState } from './../reducers/shopping.reducer';

export interface AppState {
  readonly shopping: ShoppingState
}
```

We can export the ShoppingState from our shopping.reducer.ts, as well as update any prior definitions:

```
export interface ShoppingState {
  list: ShoppingItem[],
  loading: boolean,
  error: Error
}

const initialState: ShoppingState = {
  list: [],
  loading: false,
  error: undefined
};

export function ShoppingReducer(state: ShoppingState = initialState, action: ShoppingAction) {
  // Omitted
}
```

##Reducer updates
Next up, we'll have to update our ShoppingReducer to handle the ability to switch on our new actions. Let's see how this looks:

```
export function ShoppingReducer(state: ShoppingState = initialState, action: ShoppingAction) {
  switch (action.type) {
    case ShoppingActionTypes.LOAD_SHOPPING:
      return {
        ...state,
        loading: true
      }
    case ShoppingActionTypes.LOAD_SHOPPING_SUCCESS:
      return {
        ...state,
        list: action.payload,
        loading: false
      }
    
    case ShoppingActionTypes.LOAD_SHOPPING_FAILURE: 
      return {
        ...state,
        error: action.payload,
        loading: false
      }
    
    case ShoppingActionTypes.ADD_ITEM:
      return {
        ...state,
        loading: true
      }
    case ShoppingActionTypes.ADD_ITEM_SUCCESS:
      return {
        ...state,
        list: [...state.list, action.payload],
        loading: false
      };
    case ShoppingActionTypes.ADD_ITEM_FAILURE:
      return {
        ...state,
        error: action.payload,
        loading: false
      };
    case ShoppingActionTypes.DELETE_ITEM:
      return {
        ...state,
        loading: true
      };
    case ShoppingActionTypes.DELETE_ITEM_SUCCESS:
      return {
        ...state,
        list: state.list.filter(item => item.id !== action.payload),
        loading: false
      }
    case ShoppingActionTypes.DELETE_ITEM_FAILURE:
      return {
        ...state,
        error: action.payload,
        loading: false
      };
    default:
      return state;
  }
}
```

We can make our reducer much lighter by using @ngrx/entity, and this is something we'll be investigating in a future article.

Meanwhile, our ShoppingReducer is now effectively setting loading to true each time we make an asynchronous call. Once that action either succeeds or fails, we're resetting loading and using the action.payload to drive our UI.

##Using @Effect()
We're finally at the part that looks at how to use @ngrx/effects!

Install @ngrx effects by running the following in your terminal:
```
$ npm install @ngrx/effects
```
As always, you can use the Angular CLI to set this up automatically:
```
$ ng add @ngrx/effects
```
Next, create a new file at store/effects/shopping.effects.ts and let's make an @Effect() for loadShopping:
```
import { Injectable } from '@angular/core';
import { Actions, Effect, ofType } from '@ngrx/effects';
import { map, mergeMap, catchError } from 'rxjs/operators';

import { LoadShoppingAction, ShoppingActionTypes, LoadShoppingSuccessAction, LoadShoppingFailureAction } from '../actions/shopping.actions'
import { of } from 'rxjs';
import { ShoppingService } from 'src/app/shopping.service';

@Injectable()
export class ShoppingEffects {

  @Effect() loadShopping$ = this.actions$
    .pipe(
      ofType<LoadShoppingAction>(ShoppingActionTypes.LOAD_SHOPPING),
      mergeMap(
        () => this.shoppingService.getShoppingItems()
          .pipe(
            map(data => {
              return new LoadShoppingSuccessAction(data)
            }),
            catchError(error => of(new LoadShoppingFailureAction(error)))
          )
      ),
  )

  constructor(
    private actions$: Actions,
    private shoppingService: ShoppingService
  ) { }
}
```
1. The @Effect() decorator allows @ngrx to handle the loadShopping$ Observable and this means we don't have to handle the registration/subscriptions within the Store.
2. Next, we're listening to actions and using ofType to gain access to the Action that matches LoadShoppingAction.
3. We can then use mergeMap to merge these streams and call getShoppingItems() which returns us an Observable with our shopping list items.
4. With those shopping list items in hand, we can map over the response and either dispatch a new LoadShoppingSuccessAction(data) if this was successful, otherwise use catchError to dispatch an Observable of(new LoadShoppingFailureAction(error)).

This pattern is similar for the rest of our effects:
```
import { Injectable } from '@angular/core';
import { Actions, Effect, ofType } from '@ngrx/effects';
import { map, mergeMap, catchError } from 'rxjs/operators';

import { LoadShoppingAction, ShoppingActionTypes, LoadShoppingSuccessAction, LoadShoppingFailureAction, AddItemAction, AddItemSuccessAction, AddItemFailureAction, DeleteItemAction, DeleteItemSuccessAction, DeleteItemFailureAction } from '../actions/shopping.actions'
import { of } from 'rxjs';
import { ShoppingService } from 'src/app/shopping.service';

@Injectable()
export class ShoppingEffects {

  @Effect() loadShopping$ = this.actions$
    .pipe(
      ofType<LoadShoppingAction>(ShoppingActionTypes.LOAD_SHOPPING),
      mergeMap(
        () => this.shoppingService.getShoppingItems()
          .pipe(
            map(data => {
              return new LoadShoppingSuccessAction(data)
            }),
            catchError(error => of(new LoadShoppingFailureAction(error)))
          )
      ),
  )

  @Effect() addShoppingItem$ = this.actions$
    .pipe(
      ofType<AddItemAction>(ShoppingActionTypes.ADD_ITEM),
      mergeMap(
        (data) => this.shoppingService.addShoppingItem(data.payload)
          .pipe(
            map(() => new AddItemSuccessAction(data.payload)),
            catchError(error => of(new AddItemFailureAction(error)))
          )
      )
  )
  
  @Effect() deleteShoppingItem$ = this.actions$
    .pipe(
      ofType<DeleteItemAction>(ShoppingActionTypes.DELETE_ITEM),
      mergeMap(
        (data) => this.shoppingService.deleteShoppingItem(data.payload)
          .pipe(
            map(() => new DeleteItemSuccessAction(data.payload)),
            catchError(error => of(new DeleteItemFailureAction(error)))
          )
      )
    )

  constructor(
    private actions$: Actions,
    private shoppingService: ShoppingService
  ) { }
}
```

We'll then need to add our Effects via an EffectsModule. As we're not using a feature module at this stage, we'll be using EffectsModule.forRoot() in comparison to EffectsModule.forFeature():

```
// app.module.ts
imports: [
  EffectsModule.forRoot([ShoppingEffects]),
]
```

##Hooking up our UI
Let's update our template to take advantage of our fancy Effects:
```
<div id="wrapper">
  <div [class.loading]="(loading$ | async)" id="shopping-list" *ngIf="!(error$ | async); else error">
    <div id="list">
      <h2>
        Shopping List
      </h2>

        <ul *ngIf="(shoppingItems | async); else noItems">
          <li *ngFor="let shopping of shoppingItems | async" (click)="deleteItem(shopping.id)">
            <span>{{ shopping.name }}</span>
          </li>
        </ul>

      <ng-template #noItems>
        <ul>
          <li style="max-width:250px;margin:0 auto;">No items found. Are you sure there's <em>nothing</em> you want?</li>
        </ul>
      </ng-template>
    </div>

    <form (ngSubmit)="addItem()">
      <input type="text" [(ngModel)]="newShoppingItem.name" placeholder="Item" name="itemName"/>
      <button type="submit" >+</button>
    </form>
  </div>
</div>

<ng-template #error>
  <h2>{{(error$ | async)?.message}}</h2>
</ng-template>
```
Nothing too crazy going on here. We're subscribing to selections of our state, and when we're loading we show the loading class:
```
.loading {
  box-shadow: 20px 20px 0px pink !important;
}
```
Finally, this is what our app.component.ts looks like:

```
import { Component, OnInit } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { v4 as uuid } from 'uuid';

import { AppState } from './store/models/app-state.model';
import { ShoppingItem } from './store/models/shopping-item.model';
import { AddItemAction, DeleteItemAction, LoadShoppingAction } from './store/actions/shopping.actions';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.scss']
})
export class AppComponent implements OnInit {
  
  shoppingItems: Observable<Array<ShoppingItem>>;
  loading$: Observable<Boolean>;
  error$: Observable<Error>
  newShoppingItem: ShoppingItem = { id: '', name: '' }

  constructor(private store: Store<AppState>) { }

  ngOnInit() {
    this.shoppingItems = this.store.select(store => store.shopping.list);
    this.loading$ = this.store.select(store => store.shopping.loading);
    this.error$ = this.store.select(store => store.shopping.error);

    this.store.dispatch(new LoadShoppingAction());
  }

  addItem() {
  this.newShoppingItem.id = uuid();

    this.store.dispatch(new AddItemAction(this.newShoppingItem));

    this.newShoppingItem = { id: '', name: '' };
  }

  deleteItem(id: string) {
    this.store.dispatch(new DeleteItemAction(id));
  }
}
```

When we click around our application and perform actions such as LoadShoppingAction, AddShoppingAction and DeleteShoppingAction, we'll see our Effects kick in within the Redux DevTools



