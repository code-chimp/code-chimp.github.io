---
title: "Redux Powered Notification Pipeline Pt. 2: Toasts"
description: Creating a Redux-driven notification pipeline part two - Toasts
date: 2022-10-22T00:01:01-05:00
draft: false

categories:
- Development
- User Experience

tags:
- TypeScript
- Redux
- UX
---

[Alerts][bsa] tend to be for **sticky** messages that I ensure the user is forced to actively engage and dismiss.
[Toasts][bst], on the other hand, are used for quick, __*something-happened*__ style messages - the information is there
for the user to pay attention to, or not, as the message will disappear on its own in a few seconds.

## MVP

The process for showing toast messages is nearly identical to the alerts pipeline, so I am going to start off with a
minimum-viable-product approach to verify the dispatch, show, and remove functionality.

### Interface

For this first pass I am just looking to display some text, and of course we will need a unique id for testing and
state cleanup. I chose to name the interface **IToast*Message*** over simply **IToast** as I have previously run into a
naming collision with another package, so in this instance I chose to be ultra-specific.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@interfaces/IToastMessage.ts" title="/src/@interfaces/IToastMessage.ts" >}}*
{{< highlight typescript >}}
interface IToastMessage {
  id: string;
  text?: string;
}

export default IToastMessage;
{{< /highlight >}}

### Slice

We only need actions to display it, and remove it for now:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/store/slices/toasts.ts" title="/src/store/slices/toasts.ts" >}}*
{{< highlight typescript >}}
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { v4 as uuid } from 'uuid';
import IToastMessage from '../../@interfaces/IToastMessage';
import { IStore } from '../';

const initialState: Array<IToastMessage> = [];

export const toasts = createSlice({
  name: 'toasts',
  initialState,
  reducers: {
    removeToastMessage: (state: Array<IToastMessage>, action: PayloadAction<string>) => {
      const index = state.findIndex(t => t.id === action.payload);
      if (index > -1) {
        state.splice(index, 1);
      }
    },
    addToastMessage: {
      reducer(state: Array<IToastMessage>, action: PayloadAction<IToastMessage>) {
        state.push(action.payload);
      },
      prepare(text: string): any {
        return { payload: { id: uuid(), text } };
      },
    },
  },
});

export const { addToastMessage, removeToastMessage } = toasts.actions;

export const selectToasts = (state: IStore) => state.toasts;

export default toasts.reducer;
{{< /highlight >}}

### Toasts Container

The container is really simple since [Bootstrap's live example][bsl] demonstrated a nice set of default classes
to wrap a stack of toasts:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppToasts/AppToasts.tsx" title="/src/components/app/AppToasts/AppToasts.tsx" >}}*
{{< highlight typescript >}}
import React, { FC } from 'react';
import { useAppSelector } from '../../../helpers';
import { selectToasts } from '../../../store/slices/toasts';
import Toast from './AppToast';

const AppToasts: FC = () => {
  const messages = useAppSelector(selectToasts);

  return (
    <div className="toast-container position-fixed bottom-0 end-0 p-3">
      {messages.map(message => (
        <Toast key={message.id} toastMessage={message} />
      ))}
    </div>
  );
};

export default AppToasts;
{{< /highlight >}}

### Toast Component

Here we come to the primary difference between Bootstrap's toasts and their alerts - toasts are exclusively **opt-in**
so you need to explicitly invoke the `show` method for the component. Again just the bare minimum for the toast itself:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppToasts/Toast/Toast.tsx" title="/src/components/app/AppToasts/Toast/Toast.tsx" >}}*
{{< highlight typescript "hl_lines=2 23-26" >}}
import React, { FC, useEffect, useRef } from 'react';
import { Toast as BSToast } from 'bootstrap';
import IToastMessage from '../../../../@interfaces/IToastMessage';
import { useAppDispatch } from '../../../../helpers';
import { removeToastMessage } from '../../../../store/slices/toasts';

export interface IAppToastProps {
  toastMessage: IToastMessage;
}

const Toast: FC<IAppToastProps> = ({ toastMessage }) => {
  const ref = useRef<HTMLDivElement>(null);
  const dispatch = useAppDispatch();

  useEffect(() => {
    const handleClose = () => {
      dispatch(removeToastMessage(toastMessage.id));
    };
    const el = ref.current;

    el!.addEventListener('hide.bs.toast', handleClose);

    // NOTE: unlike Alert, Toast is opt-in so you need to explicitly
    //       initialize it here.
    const toast = new BSToast(el!);
    toast.show();

    return () => {
      el!.removeEventListener('hide.bs.toast', handleClose);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <div ref={ref} className="toast" role="alert" aria-live="assertive" aria-atomic="true">
      <div className="toast-header">
        <strong className="me-auto">Alert</strong>
        <button
          type="button"
          className="btn-close"
          data-bs-dismiss="toast"
          aria-label="Close"></button>
      </div>
      <div className="toast-body">{toastMessage.text}</div>
    </div>
  );
};

export default Toast;
{{< /highlight >}}

We'll tuck the container in beside the `AppAlerts` container in the startup component:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/index.tsx" title="/src/index.tsx" >}}*
{{< highlight typescript "hl_lines=7 17" >}}
import React from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import { Provider } from 'react-redux';
import 'bootstrap';
import AppAlerts from './components/app/AppAlerts';
import AppToasts from './components/app/AppToasts';
import './styles/global.scss';
import App from './App';
import store from './store';

createRoot(document.getElementById('root') as HTMLElement).render(
  <React.StrictMode>
    <Provider store={store}>
      <BrowserRouter>
        <AppAlerts />
        <AppToasts />
        <App />
      </BrowserRouter>
    </Provider>
  </React.StrictMode>
);
{{< /highlight >}}

### Test Page

Let's add a new button to our notifications test page so that we can verify everything works:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/index.tsx" title="/src/index.tsx" >}}*
{{< highlight typescript "hl_lines=29-30 33 58-70" >}}
/* istanbul ignore file */
/* This is just an homely little demo page and is meant to be removed from a real project */
import React from 'react';
import { useAppDispatch } from '../../helpers';
import {
  addErrorAlert,
  addInfoAlert,
  addSuccessAlert,
  addWarningAlert,
} from '../../store/slices/alerts';
import { addToastMessage } from '../../store/slices/toasts';

const NotificationsDemo = () => {
  const dispatch = useAppDispatch();

  // alerts
  const handleInfoAlert = () =>
    dispatch(addInfoAlert({ title: 'Optional Title', text: 'This is purely informational' }));

  const handleSuccessAlert = () =>
    dispatch(addSuccessAlert({ text: 'very win, highly success' }));

  const handleWarningAlert = () =>
    dispatch(addWarningAlert({ text: 'Turn back, beware of tigers' }));

  const handleErrorAlert = () =>
    dispatch(addErrorAlert({ text: 'You have been eaten by a grue.' }));

  // toast
  const handleToast = () => dispatch(addToastMessage('Yay, something works!'));

  return (
    <>
      <div className="row">
        <div className="col-12">
          <h4>Alerts:</h4>
        </div>
      </div>
      <div className="row">
        <div className="col-12">
          <button onClick={handleInfoAlert} className="btn btn-info me-3">
            Info
          </button>

          <button onClick={handleSuccessAlert} className="btn btn-success me-3">
            Success
          </button>

          <button onClick={handleWarningAlert} className="btn btn-warning me-3">
            Warning
          </button>

          <button onClick={handleErrorAlert} className="btn btn-danger">
            Error
          </button>
        </div>
      </div>
      <div className="row">
        <div className="col-12">
          <h4>Toasts</h4>
        </div>
      </div>
      <div className="row">
        <div className="col-12">
          <button onClick={handleToast} className="btn btn-primary me-3">
            Test
          </button>
        </div>
      </div>
    </>
  );
};

export default NotificationsDemo;
{{< /highlight >}}

### Test Drive

After verifying that I have no linting errors it's time to start the app and push our shiny new button.
{{< figure src="first_toast.png" alt="showing a toast rendered on screen next to the redux state tree" caption="looks very MVP" >}}

After a few seconds the `removeToastMessage` was dispatched automatically by the listener and we can see that the message
has now been removed from the screen:
{{< figure src="first_toast_cleanup.png" alt="displaying no toast on the screen next to the redux dev tools showing the actions processed" caption="current store" >}}

{{< figure src="yes.gif" alt="Kip Dynamite 'yes'" caption="victory" >}}

## Adding Polish

I am going to type these similar to the the alerts, with Error, Info, Success, and Warning variants. I plan on leaving the
header text non-configurable unless a client requests it.

### Typing

Even though I am using identical values to `AlertTypes`, this component has the greatest likelihood to sprout more variants
as time goes on, so I feel it best to give the ToastMessage its own separate typing:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@enums/ToastTypes" title="/src/@enums/ToastTypes" >}}*
{{< highlight typescript >}}
enum ToastTypes {
  Error = 'danger',
  Info = 'info',
  Success = 'success',
  Warning = 'warning',
}

export default ToastTypes;
{{< /highlight >}}

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@types/ToastType" title="/src/@types/ToastType" >}}*
{{< highlight typescript >}}
import ToastTypes from '../@enums/ToastTypes';

type ToastType = ToastTypes.Error
                 | ToastTypes.Info
                 | ToastTypes.Success
                 | ToastTypes.Warning;

export default ToastType;
{{< /highlight >}}

Adding the new type to the interface:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/@interfaces/IToastMessage.ts" title="/src/@interfaces/IToastMessage.ts" >}}*
{{< highlight typescript "hl_lines=5" >}}
import ToastType from '../@types/ToastType';

interface IToastMessage {
  id: string;
  type?: ToastType;
  text?: string;
}

export default IToastMessage;
{{< /highlight >}}

### Slice Changes

Add actions for our typed toast messages and create a helper method for preparing the payload:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/store/slices/toasts.ts" title="/src/store/slices/toasts.ts" >}}*
{{< highlight typescript "hl_lines=4 9-14 26-57 61-67" >}}
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import { v4 as uuid } from 'uuid';
import IToastMessage from '../../@interfaces/IToastMessage';
import ToastTypes from '../../@enums/ToastTypes';
import { IStore } from '../';

const initialState: Array<IToastMessage> = [];

function createToastMessagePayload(toast: Partial<IToastMessage>): IToastMessage {
  return {
    ...toast,
    id: uuid(),
  };
}

export const toasts = createSlice({
  name: 'toasts',
  initialState,
  reducers: {
    removeToastMessage: (state: Array<IToastMessage>, action: PayloadAction<string>) => {
      const index = state.findIndex(t => t.id === action.payload);
      if (index > -1) {
        state.splice(index, 1);
      }
    },
    addErrorToastMessage: {
      reducer(state: Array<IToastMessage>, action: PayloadAction<IToastMessage>) {
        state.push(action.payload);
      },
      prepare(text: string): any {
        return { payload: createToastMessagePayload({ type: ToastTypes.Error, text }) };
      },
    },
    addInfoToastMessage: {
      reducer(state: Array<IToastMessage>, action: PayloadAction<IToastMessage>) {
        state.push(action.payload);
      },
      prepare(text: string): any {
        return { payload: createToastMessagePayload({ type: ToastTypes.Info, text }) };
      },
    },
    addSuccessToastMessage: {
      reducer(state: Array<IToastMessage>, action: PayloadAction<IToastMessage>) {
        state.push(action.payload);
      },
      prepare(text: string): any {
        return { payload: createToastMessagePayload({ type: ToastTypes.Success, text }) };
      },
    },
    addWarningToastMessage: {
      reducer(state: Array<IToastMessage>, action: PayloadAction<IToastMessage>) {
        state.push(action.payload);
      },
      prepare(text: string): any {
        return { payload: createToastMessagePayload({ type: ToastTypes.Warning, text }) };
      },
    },
  },
});

export const {
  addErrorToastMessage,
  addInfoToastMessage,
  addSuccessToastMessage,
  addWarningToastMessage,
  removeToastMessage,
} = toasts.actions;

export const selectToasts = (state: IStore) => state.toasts;

export default toasts.reducer;
{{< /highlight >}}

### Toast Component Changes

Adding icons and appropriate tinting to the toast header really makes it "pop":

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/components/app/AppToasts/Toast/Toast.tsx" title="/src/components/app/AppToasts/Toast/Toast.tsx" >}}*
{{< highlight typescript "hl_lines=3-11 23-27 46-69 79-87" >}}
import React, { FC, useEffect, useRef } from 'react';
import { Toast as BSToast } from 'bootstrap';
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import {
  faCircleCheck,
  faCircleInfo,
  faCircleXmark,
  faTriangleExclamation,
  IconDefinition,
} from '@fortawesome/free-solid-svg-icons';
import ToastTypes from '../../../../@enums/ToastTypes';
import IToastMessage from '../../../../@interfaces/IToastMessage';
import { useAppDispatch } from '../../../../helpers';
import { removeToastMessage } from '../../../../store/slices/toasts';

export interface IAppToastProps {
  toastMessage: IToastMessage;
}

const Toast: FC<IAppToastProps> = ({ toastMessage }) => {
  const ref = useRef<HTMLDivElement>(null);
  const dispatch = useAppDispatch();
  let icon: IconDefinition;
  let headerText: string;
  // we will add our tint class to this based on type then just
  // `string.join` the array for the header's "className"
  let headerClasses = ['toast-header', ' bg-opacity-25'];

  useEffect(() => {
    const handleClose = () => {
      dispatch(removeToastMessage(toastMessage.id));
    };
    const el = ref.current;

    el!.addEventListener('hidden.bs.toast', handleClose);

    const toast = new BSToast(el!);
    toast.show();

    return () => {
      el!.removeEventListener('hidden.bs.toast', handleClose);
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  switch (toastMessage.type) {
    case ToastTypes.Error:
      icon = faCircleXmark;
      headerText = 'Error';
      headerClasses = [...headerClasses, 'bg-danger', 'text-danger'];
      break;

    case ToastTypes.Success:
      icon = faCircleCheck;
      headerText = 'Success';
      headerClasses = [...headerClasses, 'bg-success', 'text-success'];
      break;

    case ToastTypes.Warning:
      icon = faTriangleExclamation;
      headerText = 'Warning';
      headerClasses = [...headerClasses, 'bg-warning', 'text-warning'];
      break;

    default:
      icon = faCircleInfo;
      headerText = 'Information';
      headerClasses = [...headerClasses, 'bg-info', 'text-primary'];
  }

  return (
    <div
      ref={ref}
      data-testid={`toast-${toastMessage.id}`}
      className="toast"
      role="alert"
      aria-live="assertive"
      aria-atomic="true">
      <div className={headerClasses.join(' ')}>
        <FontAwesomeIcon
          data-testid={`toast-${toastMessage.id}-icon`}
          className="flex-shrink-0 me-2"
          icon={icon}
        />
        <strong data-testid={`toast-${toastMessage.id}-header`} className="me-auto">
          {headerText}
        </strong>
        <button
          type="button"
          data-testid={`toast-${toastMessage.id}-close`}
          className="btn-close"
          data-bs-dismiss="toast"
          aria-label="Close"></button>
      </div>
      <div data-testid={`toast-${toastMessage.id}-body`} className="toast-body">
        {toastMessage.text}
      </div>
    </div>
  );
};

export default Toast;
{{< /highlight >}}

### Test Page Changes

Add some new buttons for the new toast actions:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/pages/NotificationsDemo/NotificationsDemo.tsx" title="/src/pages/NotificationsDemo/NotificationsDemo.tsx" >}}*
{{< highlight typescript "hl_lines=11-16 35-43 78-92" >}}
/* istanbul ignore file */
/* This is just an homely little demo page and is meant to be removed from a real project */
import React from 'react';
import { useAppDispatch } from '../../helpers';
import {
  addErrorAlert,
  addInfoAlert,
  addSuccessAlert,
  addWarningAlert,
} from '../../store/slices/alerts';
import {
  addErrorToastMessage,
  addInfoToastMessage,
  addSuccessToastMessage,
  addWarningToastMessage,
} from '../../store/slices/toasts';

const NotificationsDemo = () => {
  const dispatch = useAppDispatch();

  // alerts
  const handleInfoAlert = () =>
    dispatch(addInfoAlert({ title: 'Optional Title', text: 'This is purely informational' }));

  const handleSuccessAlert = () =>
    dispatch(addSuccessAlert({ text: 'very win, highly success' }));

  const handleWarningAlert = () =>
    dispatch(addWarningAlert({ text: 'Turn back, beware of tigers' }));

  const handleErrorAlert = () =>
    dispatch(addErrorAlert({ text: 'You have been eaten by a grue.' }));

  // toasts
  const handleInfoToast = () => dispatch(addInfoToastMessage('This is purely informational'));

  const handleSuccessToast = () =>
    dispatch(addSuccessToastMessage('You succeeded, give yourself a prize'));

  const handleWarningToast = () =>
    dispatch(addWarningToastMessage('Highway to the danger zone'));

  const handleErrorToast = () => dispatch(addErrorToastMessage('The roof is on fire'));

  return (
    <>
      <div className="row">
        <div className="col-12">
          <h4>Alerts:</h4>
        </div>
      </div>
      <div className="row">
        <div className="col-12">
          <button onClick={handleInfoAlert} className="btn btn-info me-3">
            Info
          </button>

          <button onClick={handleSuccessAlert} className="btn btn-success me-3">
            Success
          </button>

          <button onClick={handleWarningAlert} className="btn btn-warning me-3">
            Warning
          </button>

          <button onClick={handleErrorAlert} className="btn btn-danger">
            Error
          </button>
        </div>
      </div>
      <div className="row">
        <div className="col-12">
          <h4>Toasts</h4>
        </div>
      </div>
      <div className="row">
        <div className="col-12">
          <button onClick={handleInfoToast} className="btn btn-info me-3">
            Info
          </button>

          <button onClick={handleSuccessToast} className="btn btn-success me-3">
            Success
          </button>

          <button onClick={handleWarningToast} className="btn btn-warning me-3">
            Warning
          </button>

          <button onClick={handleErrorToast} className="btn btn-danger">
            Error
          </button>
        </div>
      </div>
    </>
  );
};

export default NotificationsDemo;
{{< /highlight >}}

### Final Test

The page may be ugly, but the toasts look good!
{{< figure src="final_toasts.png" alt="displaying four toast messages on the screen with appropriate icons and coloring" caption="Yeah, Toast!" >}}


## Conclusion

I am pretty happy with the results and feel this is good enough for a project starter. Team members on projects that I
have added these pipelines to seem to like it, and the designers were not completely horrified. Unit tests will be in
[the repository][van] if you would like some good code coverage to go with the sample code. As always feel free to drop
me a line if you see anywhere that could use some improvement.

[van]: https://github.com/code-chimp/vanilla-redux-template 'My base template for new Redux projects'
[bsa]: https://getbootstrap.com/docs/5.2/components/alerts/ 'Provide contextual feedback messages for typical user actions'
[bst]: https://getbootstrap.com/docs/5.2/components/toasts/ 'Lightweight push notifications'
[bsl]: https://getbootstrap.com/docs/5.2/components/toasts/#live-example 'toast live example'
