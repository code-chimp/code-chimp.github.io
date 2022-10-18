---
title: "Redux Powered Notification Pipeline Pt. 1: Alerts"
description: Creating a Redux-driven notification pipeline part one - Alerts
date: 2022-10-20T00:01:01-05:00
draft: false

categories:
- Development
- User Experience

tags:
- TypeScript
- Redux
- UX
---

## Motivation

Having your application relay relevant, timely feedback to your user is critical - especially if it relies on a great
deal of asynchronous interactions. Two of the standard ways of delivering feedback are through the use of [alerts][bsa]
and [toast messages][bst]. To avoid a lot of boilerplate markup popping up all over the project I wanted to make it as
simple for a developer as just dispatching an action such as `dispatch(errorAlert('your call cannot be completed as dialed');`
and have the alert or toast appear on the screen. I am using [Bootstrap][bs] for this project but the same concept should
translate to other frameworks such as [Ant Design][antd], [Material UI][mui], or [Foundation for Sites][zurb].

What follows is a short replay of how I implemented an asynchronous alerts pipeline in
{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/" title="my Redux template project" >}}, and
one or more of the issues, and their subsequent solutions, I ran into during the process.

## Implementation

[Bootstrap][bs] was added to the project via a global stylesheet:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/styles/global.scss" title="/src/styles/global.scss" >}}*
{{< highlight scss >}}
@import '~bootstrap/scss/bootstrap';
{{< /highlight >}}

and then included along with their JavaScript plugin in the project's startup file:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/index.tsx" title="/src/index.tsx" >}}*
{{< highlight typescript "hl_lines=5 6" >}}
import React from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import 'bootstrap';
import './styles/global.scss';
import App from './App';
import store from './store';

createRoot(document.getElementById('root') as HTMLElement).render(
  <React.StrictMode>
    <Provider store={store}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </Provider>
  </React.StrictMode>
);
{{< /highlight >}}

As we can see from their [documentation][bsax] there are several different styles of alert available from their framework
that are powered by the addition of a custom class such as `alert-success` or `alert-info`.
{{< figure src="alert_examples.png" alt="Bootstrap alert examples" >}}

### Typing

For this exercise I am only concerned with a specific subset of these, so I created an enumeration with an associated type
that allows me to utilize their string-based names in a more type safe manner. This has an additional benefit of protecting
me from any inevitable typos (i.e. `btn-sucess` `btn-daanger` etc.).

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@enums/AlertTypes.ts" title="/src/@enums/AlertTypes.ts" >}}*
{{< highlight typescript >}}
enum AlertTypes {
  Error = 'danger',
  Info = 'info',
  Success = 'success',
  Warning = 'warning',
}

export default AlertTypes;
{{< /highlight >}}

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@types/AlertType.ts" title="/src/@types/AlertType.ts" >}}*
{{< highlight typescript >}}
import AlertTypes from '../@enums/AlertTypes';

type AlertType = AlertTypes.Error
                 | AlertTypes.Info
                 | AlertTypes.Success
                 | AlertTypes.Warning;

export default AlertType;
{{< /highlight >}}

Before we can begin to flesh out a slice for our alerts we need to decide the shape of the alert object that we will be
placing in the store. [The Bootstrap documentation][bsad] shows some good examples of the kind of content you can put
inside of a Bootstrap Alert, so for this first pass I am just going with the alert _text_, _type_, and an optional _title_:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@interfaces/IAlert.ts" title="/src/@interfaces/IAlert.ts" >}}*
{{< highlight typescript >}}
import AlertType from '../@types/AlertType';

interface IAlert {
  type: AlertType;
  text: string;
  title?: string;
}

export default IAlert;
{{< /highlight >}}

### Slice

For the slice I am thinking that we just need to hold and array of our `IAlert` shaped objects that can be picked up by a reactive
component watching for changes to the `store.alerts`. Since we are actually *adding* to an array of alert messages I
believe a more accurate action name should be `dispatch(addErrorAlert(...` as opposed to my initial thought of
`dispatch(errorAlert(...` to better reflect how we are actually changing the store. Let's start off with just two
actions in order to assess the validity of this approach. This felt like a good first pass:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/store/slices/alerts.ts" title="/src/store/slices/alerts.ts" >}}*
{{< highlight typescript >}}
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import IAlert from '../../@interfaces/IAlert';
import AlertTypes from '../../@enums/AlertTypes';
import { IStore } from '../';

const initialState: Array<IAlert> = [];

export const alerts = createSlice({
  name: 'alerts',
  initialState,
  reducers: {
    addInfoAlert: (state: Array<IAlert>, action: PayloadAction<IAlert>) => {
      state.push({
        ...action.payload,
        type: AlertTypes.Info,
      });
    },
    addSuccessAlert: (state: Array<IAlert>, action: PayloadAction<IAlert>) => {
      state.push({ ...action.payload, type: AlertTypes.Success });
    },
  },
});

export const { addInfoAlert, addSuccessAlert } = alerts.actions;

export const selectAlerts = (state: IStore) => state.alerts;

export default alerts.reducer;
{{< /highlight >}}

### Container and Alert Components

Now we just need a functional container component to watch our store for new alerts:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppAlerts/AppAlerts.tsx" title="/src/components/app/AppAlerts/AppAlerts.tsx" >}}*
{{< highlight typescript >}}
import React, { FC } from 'react';
import { useAppSelector } from '../../../helpers';
import { selectAlerts } from '../../../store/slices/alerts';
import styles from './AppAlerts.module.scss';
import Alert from './Alert';
import IAlert from '../../../@interfaces/IAlert';

const AppAlerts: FC = () => {
  const alerts = useAppSelector(selectAlerts);

  return (
    <div className={styles.wrapper}>
      <div className={styles.alertsCol}>
        {alerts.map((alert: IAlert, index: number) => (
          // yes `index` is a bit greasy, but give me a minute
          <Alert key={index} alert={alert} />
        ))}
      </div>
    </div>
  );
};

export default AppAlerts;
{{< /highlight >}}

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppAlerts/Alert/Alert.tsx" title="/src/components/app/AppAlerts/Alert/Alert.tsx" >}}*
{{< highlight typescript >}}
import React, { FC } from 'react';
import IAlert from '../../../../@interfaces/IAlert';

export interface IAlertProps {
  alert: IAlert;
}

const Alert: FC<IAlertProps> = ({ alert }) => {
  return (
    <div
      className={`alert alert-${alert.type} alert-dismissible fade show d-flex align-items-center`}
      role="alert">
      <div>
        {alert.title ? <h5 className="mb-0">{alert.title}</h5> : null}
        {alert.text}
      </div>
      <button
        type="button"
        className="btn-close"
        data-bs-dismiss="alert"
        aria-label="Close"></button>
    </div>
  );
};

export default Alert;
{{< /highlight >}}

Since we want this to be available from anywhere in the application I am going to put it next to our main `App` component.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/index.tsx" title="/src/index.tsx" >}}*
{{< highlight typescript "hl_lines=15" >}}
import React from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import 'bootstrap';
import AppAlerts from './components/app/AppAlerts';
import './styles/global.scss';
import App from './App';
import store from './store';

createRoot(document.getElementById('root') as HTMLElement).render(
  <React.StrictMode>
    <Provider store={store}>
      <BrowserRouter>
        <AppAlerts />
        <App />
      </BrowserRouter>
    </Provider>
  </React.StrictMode>
);
{{< /highlight >}}

### Test Page

And now an ugly little screen to test our logic:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/pages/NotificationsDemo/NotificationsDemo.tsx" title="/src/pages/NotificationsDemo/NotificationsDemo.tsx" >}}*
{{< highlight typescript >}}
import React from 'react';
import { useAppDispatch } from '../../helpers';
import { addInfoAlert, addSuccessAlert } from '../../store/slices/alerts';

const NotificationsDemo = () => {
  const dispatch = useAppDispatch();

  const handleInfoAlert = () =>
    dispatch(addInfoAlert({ title: 'Optional Title', text: 'This is purely informational' }));

  const handleSuccessAlert = () =>
    dispatch(addSuccessAlert({ text: 'very win, highly success' }));

  return (
    <div className="row">
      <div className="col-12">
        <button onClick={handleInfoAlert} className="btn btn-info me-3">
          Info
        </button>

        <button onClick={handleSuccessAlert} className="btn btn-success me-3">
          Success
        </button>
      </div>
    </div>
  );
};

export default NotificationsDemo;
{{< /highlight >}}

#### ...and, our first error

{{< figure src="first_error.png" alt="linting error on the dispatched action's argument type" caption="my first error (today)" >}}
The excellent TypeScript linter has alerted me to a problem. The way that I have designed the actions they require the entire `IAlert`
interface including the type. I was really hoping to abstract that implementation detail away from the developer so let's
see what can be done about that.

The solution was actually fairly easy. [Redux Toolkit][rtk] provides an optional [prepare callback][prep] argument to allow
you to do additional processing to the incoming action's payload before passing it on to the actual reducer. Here I
configure the action `prepare` method to accept a partial `IAlert` which we then fill in the missing `AlertType`
field before passing the now complete `IAlert` to the reducer.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/store/slices/alerts.ts" title="/src/store/slices/alerts.ts" >}}*
{{< highlight typescript "hl_lines=16-18 24-26" >}}
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import IAlert from '../../@interfaces/IAlert';
import AlertTypes from '../../@enums/AlertTypes';
import { IStore } from '../';

const initialState: Array<IAlert> = [];

export const alerts = createSlice({
  name: 'alerts',
  initialState,
  reducers: {
    addInfoAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: { ...alert, type: AlertTypes.Info } };
      },
    },
    addSuccessAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: { ...alert, type: AlertTypes.Success } };
      },
    },
  },
});

export const { addInfoAlert, addSuccessAlert } = alerts.actions;

export const selectAlerts = (state: IStore) => state.alerts;

export default alerts.reducer;
{{< /highlight >}}

### First Test Drive

ESLint is happy and we can now fire up the app and see what damage we have done.
{{< figure src="first_alerts.png" alt="showing two alerts on the screen" caption="hooray!" >}}

Looks like the components are accurately reflecting the current store:
{{< figure src="first_alerts_state.png" alt="showing two alerts on the screen next to the redux dev tools showing the current state tree" caption="current store" >}}

Most of us have experienced the feeling, right?
{{< figure src="smartest_dev_alive.png" alt="dumb meme joking that I am a bronze medalist who thinks they got the gold" caption="Hecks YEAH!!1!" >}}

#### ...oh, a second design flaw

Let's click the close buttons to get rid of these before we take a victory lap - but what's this? The state still contains our alerts that we closed.
{{< figure src="second_error.png" alt="no alerts on screen, but still in the state tree" caption="why are they still there?" >}}

The really bad thing is that the alerts functionality still works and will add new alerts and not display the old ones,
but that still leaves state that is no longer relevant in our store. Of course this is not optimal so we need to fix it.
{{< figure src="pollution.png" alt="bloated state tree with stale information" caption="bloated with stale information" >}}

### Removing Stale Messages

We need to add a unique identifier for each alert so that we can easily locate it and remove it from state via a new
action. Let's add [uuid][uuid] to help with this:
{{< highlight shell >}}
yarn add uuid
yarn add --dev @types/uuid

# or if you prefer

npm i uuid
npm i -D @types/uuid
{{< /highlight >}}

Add an `id` property to our alert interface:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@interfaces/IAlert.ts" title="/src/@interfaces/IAlert.ts" >}}*
{{< highlight typescript "hl_lines=4" >}}
import AlertType from '../@types/AlertType';

interface IAlert {
  id: string;
  type: AlertType;
  text: string;
  title?: string;
}

export default IAlert;
{{< /highlight >}}

We will generate the id and add it to the alert in the action's `prepare` callback. Since this is starting to add some
repetitive logic let us go ahead and pull the payload preparation logic out to a helper method.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/store/slices/alerts.ts" title="/src/store/slices/alerts.ts" >}}*
{{< highlight typescript "hl_lines=2 10-16 27 35" >}}
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { v4 as uuid } from 'uuid';
import AlertTypes from '../../@enums/AlertTypes';
import AlertType from '../../@types/AlertType';
import IAlert from '../../@interfaces/IAlert';
import { IStore } from '../';

const initialState: Array<IAlert> = [];

const createAlertPayload = (type: AlertType, alert: Partial<IAlert>): IAlert => {
  return {
    ...alert,
    id: uuid(),
    type,
  };
};

export const alerts = createSlice({
  name: 'alerts',
  initialState,
  reducers: {
    addInfoAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Info, alert) };
      },
    },
    addSuccessAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Success, alert) };
      },
    },
  },
});

export const { addInfoAlert, addSuccessAlert } = alerts.actions;

export const selectAlerts = (state: IStore) => state.alerts;

export default alerts.reducer;
{{< /highlight >}}

Back in the slice let's add an action to remove a specified alert from the array.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/store/slices/alerts.ts" title="/src/store/slices/alerts.ts" >}}*
{{< highlight typescript "hl_lines=22-27 48" >}}
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { v4 as uuid } from 'uuid';
import AlertTypes from '../../@enums/AlertTypes';
import AlertType from '../../@types/AlertType';
import IAlert from '../../@interfaces/IAlert';
import { IStore } from '../';

const initialState: Array<IAlert> = [];

const createAlertPayload = (type: AlertType, alert: Partial<IAlert>): IAlert => {
  return {
    ...alert,
    id: uuid(),
    type,
  };
};

export const alerts = createSlice({
  name: 'alerts',
  initialState,
  reducers: {
    removeAlert: (state: Array<IAlert>, action: PayloadAction<string>) => {
      const index = state.findIndex(a => a.id === action.payload);
      if (index > -1) {
        state.splice(index, 1);
      }
    },
    addInfoAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Info, alert) };
      },
    },
    addSuccessAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Success, alert) };
      },
    },
  },
});

// don't forget to export the new action
export const { addInfoAlert, addSuccessAlert, removeAlert } = alerts.actions;

export const selectAlerts = (state: IStore) => state.alerts;

export default alerts.reducer;
{{< /highlight >}}


Now we can fix that greasy key in the alerts container:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppAlerts/AppAlerts.tsx" title="/src/components/app/AppAlerts/AppAlerts.tsx" >}}*
{{< highlight typescript  "hl_lines=15" >}}
import React, { FC } from 'react';
import { useAppSelector } from '../../../helpers';
import { selectAlerts } from '../../../store/slices/alerts';
import styles from './AppAlerts.module.scss';
import Alert from './Alert';
import IAlert from '../../../@interfaces/IAlert';

const AppAlerts: FC = () => {
  const alerts = useAppSelector(selectAlerts);

  return (
    <div className={styles.wrapper}>
      <div className={styles.alertsCol}>
        {alerts.map((alert: IAlert, index: number) => (
          <Alert key={alert.id} alert={alert} />
        ))}
      </div>
    </div>
  );
};

export default AppAlerts;
{{< /highlight >}}

So now we know each alert will have its own unique identifier, and that we have an action to remove an alert object out
of the state array using the identifier - the question arises where is the best place to actually dispatch the `removeAlert`
action from? Back in [the Bootstrap Alert documentation][bsevt] it shows a pair of events, `close.bs.alert` and `closed.bs.alert`
that we could attach an event listener to. Initially I chose `closed.bs.alert` thinking that the component might unexpectedly
disappear from the user's screen while animations were still in progress. After running into intermittent errors I realized
that I was causing errors while attempting to unbind an event listener from a div that Bootstrap had already destroyed.
By switching the event listener to `close.bs.alert` it appears to give us enough time to unbind our listener without
throwing DOMNode errors. To attach our listener to this exact alert instance we will need a reference to the `div` just
created which we can obtain from the [useRef][ref] hook. We can be certain that the `ref.current` is populated with the
desired DOM reference by wrapping it inside a single-fire **useEffect**.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppAlerts/Alert/Alert.tsx" title="/src/components/app/AppAlerts/Alert/Alert.tsx" >}}*
{{< highlight typescript  "hl_lines=1 3 4 11 12 14-30 34" >}}
import React, { FC, useEffect, useRef } from 'react';
import IAlert from '../../../../@interfaces/IAlert';
import { removeAlert } from '../../../../store/slices/alerts';
import { useAppDispatch } from '../../../../helpers';

export interface IAlertProps {
  alert: IAlert;
}

const Alert: FC<IAlertProps> = ({ alert }) => {
  const ref = useRef<HTMLDivElement>(null);
  const dispatch = useAppDispatch();

  useEffect(() => {
    // do not inline this so that we have a reference to use for subscribing and unsubscribing
    const handleClose = () => {
      dispatch(removeAlert(alert.id));
    };
    // a reference to `this` exact Alert instance
    const el = ref.current;

    el!.addEventListener('close.bs.alert', handleClose);

    // to ensure that we do not leave a zombie process hanging around in memory
    //   this method will fire when the component unmounts
    return () => {
      el!.removeEventListener('close.bs.alert', handleClose);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <div
      ref={ref}
      className={`alert alert-${alert.type} alert-dismissible fade show d-flex align-items-center`}
      role="alert">
      <div>
        {alert.title ? <h5 className="mb-0">{alert.title}</h5> : null}
        {alert.text}
      </div>
      <button
        type="button"
        className="btn-close"
        data-bs-dismiss="alert"
        aria-label="Close"></button>
    </div>
  );
};

export default Alert;
{{< /highlight >}}

### Second Test Drive

This should be all of the moving pieces required to keep our state tidy. Clicking both buttons so we have a duplicate
scenario to the first time through:
{{< figure src="second_alerts.png" alt="showing two alerts on the screen next to state tree" caption="same setup" >}}

Let's close the first info alert and check the state:
{{< figure src="second_alerts_success.png" alt="showing one alert on the screen next to now modified state tree" caption="it is gone!" >}}

### Finishing Touches

Now that we know everything works as intended we can implement the remaining actions missing from the alerts slice and
call it day:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/store/slices/alerts.ts" title="/src/store/slices/alerts.ts" >}}*
{{< highlight typescript "hl_lines=28-35 52-59 63" >}}
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { v4 as uuid } from 'uuid';
import AlertTypes from '../../@enums/AlertTypes';
import AlertType from '../../@types/AlertType';
import IAlert from '../../@interfaces/IAlert';
import { IStore } from '../';

const initialState: Array<IAlert> = [];

const createAlertPayload = (type: AlertType, alert: Partial<IAlert>): IAlert => {
  return {
    ...alert,
    id: uuid(),
    type,
  };
};

export const alerts = createSlice({
  name: 'alerts',
  initialState,
  reducers: {
    removeAlert: (state: Array<IAlert>, action: PayloadAction<string>) => {
      const index = state.findIndex(a => a.id === action.payload);
      if (index > -1) {
        state.splice(index, 1);
      }
    },
    addErrorAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Error, alert) };
      },
    },
    addInfoAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Info, alert) };
      },
    },
    addSuccessAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Success, alert) };
      },
    },
    addWarningAlert: {
      reducer(state: Array<IAlert>, action: PayloadAction<IAlert>) {
        state.push(action.payload);
      },
      prepare(alert: Partial<IAlert>): any {
        return { payload: createAlertPayload(AlertTypes.Warning, alert) };
      },
    },
  },
});

export const { addErrorAlert, addInfoAlert, addSuccessAlert, addWarningAlert, removeAlert } =
  alerts.actions;

export const selectAlerts = (state: IStore) => state.alerts;

export default alerts.reducer;
{{< /highlight >}}


## Bonus Round

{{< figure src="brians_flair.gif" alt="brian from office space showing his flair" caption="It's FLAIR TIME!" >}}

While our alerts look nice why don't we go the extra few feet and add some [FontAwesome][fa] icons for a little flair.
Here I am just taking what was shown [in the docs][icon] and swapping in [FontAwesome][fa]'s icons:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppAlerts/Alert/Alert.tsx" title="/src/components/app/AppAlerts/Alert/Alert.tsx" >}}*
{{< highlight typescript  "hl_lines=2-10 14 38-55 63" >}}
import React, { FC, useEffect, useRef } from 'react';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { IconDefinition } from '@fortawesome/free-regular-svg-icons';
import {
  faCircleCheck,
  faCircleInfo,
  faCircleQuestion,
  faCircleXmark,
  faTriangleExclamation,
} from '@fortawesome/free-solid-svg-icons';
import IAlert from '../../../../@interfaces/IAlert';
import { removeAlert } from '../../../../store/slices/alerts';
import { useAppDispatch } from '../../../../helpers';
import AlertTypes from '../../../../@enums/AlertTypes';

export interface IAlertProps {
  alert: IAlert;
}

const Alert: FC<IAlertProps> = ({ alert }) => {
  const ref = useRef<HTMLDivElement>(null);
  const dispatch = useAppDispatch();

  useEffect(() => {
    const handleClose = () => {
      dispatch(removeAlert(alert.id));
    };
    const el = ref.current;

    el!.addEventListener('close.bs.alert', handleClose);

    return () => {
      el!.removeEventListener('close.bs.alert', handleClose);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  let icon: IconDefinition;

  switch (alert.type) {
    case AlertTypes.Error:
      icon = faCircleXmark;
      break;
    case AlertTypes.Success:
      icon = faCircleCheck;
      break;
    case AlertTypes.Warning:
      icon = faTriangleExclamation;
      break;
    case AlertTypes.Info:
      icon = faCircleInfo;
      break;
    default:
      icon = faCircleQuestion;
  }

  return (
    <div
      ref={ref}
      data-testid={`alert-${alert.id}`}
      className={`alert alert-${alert.type} alert-dismissible fade show d-flex align-items-center`}
      role="alert">
      <FontAwesomeIcon className="flex-shrink-0 me-2" icon={icon} />
      <div>
        {alert.title ? <h5 className="mb-0">{alert.title}</h5> : null}
        {alert.text}
      </div>
      <button
        type="button"
        className="btn-close"
        data-bs-dismiss="alert"
        aria-label="Close"></button>
    </div>
  );
};

export default Alert;
{{< /highlight >}}

I feel the icon adds a little extra polish for some miniscule extra effort:
{{< figure src="final_alerts.png" alt="showing four variations of alert message with an svg icon based on alert type" caption="final result" >}}

## Conclusion

I am going to wrap up this session by writing some unit tests around the new code which you will be able to see in
[the repository][van]. I promise that part two, where we implement the [toast][bst]'s pipeline, will be a lot shorter of an
article. The process was nearly identical to developing alerts with a couple of minor twists which I will call out. As
always feel free to drop me a line if you see anywhere that could use some improvement.


[van]: https://github.com/code-chimp/vanilla-redux-template 'My base template for new Redux projects'
[bs]: https://getbootstrap.com/ 'Powerful front-end toolkit'
[antd]: https://ant.design/docs/react/introduce 'Popular UI toolkit'
[mui]: https://mui.com/ 'A comprehensive suite of UI tools to help you ship new features faster'
[zurb]: https://get.foundation/sites.html 'A wide range of modular and flexible components with an eye toward accessibility'
[bsa]: https://getbootstrap.com/docs/5.2/components/alerts/ 'Provide contextual feedback messages for typical user actions'
[bsax]: https://getbootstrap.com/docs/5.2/components/alerts/#examples 'Examples of Bootstrap alert styles'
[bsad]: https://getbootstrap.com/docs/5.2/components/alerts/#additional-content 'Additional content in the Alert body'
[bst]: https://getbootstrap.com/docs/5.2/components/toasts/ 'Lightweight push notifications'
[icon]: https://getbootstrap.com/docs/5.2/components/alerts/#icons 'Adding Bootstrap icons'
[prep]: https://redux-toolkit.js.org/api/createaction/#using-prepare-callbacks-to-customize-action-contents 'Using Prepare Callbacks to Customize Action Contents'
[rtk]: https://redux-toolkit.js.org/ 'official, opinionated, batteries-included toolset for efficient Redux development'
[uuid]: https://github.com/uuidjs/uuid 'For the creation of RFC4122 UUIDs'
[bsevt]: https://getbootstrap.com/docs/5.2/components/alerts/#events 'Fired when the alert has been closed and CSS transitions have completed.'
[ref]: https://reactjs.org/docs/hooks-reference.html#useref 'returns a mutable ref object whose .current property is initialized to the passed argument i.e. like a DOM node'
[fa]: https://fontawesome.com/ "The internet's icon library"
