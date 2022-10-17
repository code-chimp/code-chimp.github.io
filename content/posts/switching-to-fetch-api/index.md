---
title: "Switching to Fetch API"
description: Eliminating a package dependency by switching from Axios to the Fetch API
date: 2022-10-16T19:58:57-05:00
draft: false

tags:
- TypeScript
- Optimization
- API
- Native API
- Fetch
- Axios
---

I have finally made the decision to let go of one of my favorite NPM packages, [Axios][ax], in favor of modern browsers'
[Fetch API][ftch]. I want to be clear up front - **I find nothing wrong with Axios**, it is an extremely high quality
package and a natural progression having used [Angular][ng]'s http service that it was originally based upon. I will
likely still rely on Axios in [NodeJS][node] projects, but times change and it now seems a bit redundant in front-end
client applications.

There are really only a few factors that are prompting me to make this change:

1. There is no longer a need for me to support legacy browsers
2. A need to get back exactly what was being sent by the server without any additional overhead
3. No longer seeing the necessity of such a package on the client when the native browser API is more than sufficient

## Before

So here is a typical, albeit contrived, previous use of Axios in a front-end service on my
{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/" title="Redux template project" >}}:

file: */src/services/user/UsersApi.ts*
{{< highlight typescript >}}
import { AxiosResponse } from 'axios';
import IUser from '../../@interfaces/IUser';
import { getAxiosInstance } from '../baseService';
import { unwrapServiceError } from '../../util/service';

axios.defaults.baseURL = 'https://jsonplaceholder.typicode.com/';

export const fetchUsers = async (): Promise<Array<IUser>> => {
  try {
    const axios = getAxiosInstance();
    const response: AxiosResponse = await axios.get('/users');

    return response.data;
  } catch (e) {
    throw unwrapServiceError('UsersApi.fetchUsers', e);
  }
};
{{< /highlight >}}

Along with the factory method to create the Axios instance:

file: */src/services/baseService.ts*
{{< highlight typescript >}}
import axios, { AxiosInstance, AxiosRequestConfig } from 'axios';
import { config as appConfig } from '../config';

export const getAxiosInstance = (token?: string): AxiosInstance => {
  const getConfig = (token?: string): AxiosRequestConfig => {
    const config: AxiosRequestConfig = {
      baseURL: appConfig.apiUrl,
      headers: {
        'Cache-Control': 'no-cache',
        'Content-Type': 'application/json',
        Pragma: 'no-cache',
      },
    };

    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    return config;
  };

  const config = getConfig(token);
  const instance = axios.create(config);

  // NOTE: Wire interceptors here.

  return instance;
};
{{< /highlight >}}

And finally a utility I wrote to normalize Axios errors along with other types of framework errors:

file: */src/util/service.ts*
{{< highlight typescript >}}
export const unwrapServiceError = (serviceMethod: string, error: any): AxiosError | Error => {
  if (error.response) {
    // utilize more robust AxiosError
    const e = error as AxiosError;

    if (e.response!.status === HttpStatusCodes.BadRequest && !e.response!.data?.message) {
      e.response!.data = e.response!.data || {};
      e.response!.data.message = `${serviceMethod}: Not Found`;
    }

    return e;
  }

  return new Error(`${serviceMethod} error: ${error.message}`);
};
{{< /highlight >}}

This pattern and code nearly verbatim has served me well for many, many projects.

## The Rewrite

My process for switching to the native Fetch API started with a visit to the
{{< newtabref href="https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch#supplying_request_options" title="MDN documentation" >}}
pertaining to passing body, options, headers etc. to a fetch request. It was pretty obvious that it would not be too
difficult of a process to create a helper method to do something similar to `axios.create`. I opted just to
create the request body and explicitly pass it to **fetch** in the service so that it is clear that we are using the
native API.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/helpers/service.ts" title="/src/helpers/service.ts" >}}*
{{< highlight typescript >}}
import { ACCESS_ERROR, GENERIC_SERVICE_ERROR } from '../constants';

// for the demo this is in a `.env` file that Create-React-App is auto-wired to pick up
const apiUri = process.env.REACT_APP_API_URI;

function createRequest(method: FetchRequestType) {
  return (url: string, body?: any, token?: string): Request => {
    const headers: Headers = new Headers({
      'Cache-Control': 'no-cache',
      Pragma: 'no-cache',
    });

    if (body) {
      headers.append('Content-Type', 'application/json');
    }

    if (token) {
      headers.append('Authorization', `Bearer ${token}`);
    }

    const init: RequestInit = {
      method,
      mode: 'cors',
      cache: 'no-cache',
      headers,
    };

    if (token) {
      init.credentials = 'include';
    }

    if (body) {
      init.body = JSON.stringify(body);
    }

    return new Request(`${apiUri}${url}`, init);
  };
}

{{< /highlight >}}

Maybe a little more verbose, but I opted for clarity when naming my exported helper functions:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/helpers/service.ts" title="/src/helpers/service.ts" >}}*
{{< highlight typescript >}}
export const createGetRequest = createRequest(FetchMethods.Get);
export const createPatchRequest = createRequest(FetchMethods.Patch);
export const createPostRequest = createRequest(FetchMethods.Post);
export const createPutRequest = createRequest(FetchMethods.Put);
export const createDeleteRequest = createRequest(FetchMethods.Delete);
{{< /highlight >}}

Since there is a lot of repetitive boilerplate around processing a fetch response I opted for another helper method and decided
that it was a good place to take advantage of TypeScript's generics for stronger typing when processing the response's payload.
Since I have run into API's that return compound responses, similar to an AxiosResponse with the payload coming in a `data`
field, I check for those here so I can dig the value I am looking for seamlessly out of the payload. This is probably a
good spot for fine-tuning to your specific needs.

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/helpers/service.ts" title="/src/helpers/service.ts" >}}*
{{< highlight typescript >}}
export async function processApiResponse<Type>(response: Response): Promise<Type> {
  if (response.ok) {
    const payload = await response.json();

    // see if we have a compound API response
    if (payload.status && (payload.data || payload.errors)) {
      if (!payload.success) {
        throw payload;
      }

      return payload.data as Type;
    }

    return payload as Type;
  }

  if (
    response.status === HttpStatusCodes.Unauthorized ||
    response.status === HttpStatusCodes.Forbidden
  ) {
    throw new Error(ACCESS_ERROR);
  }

  throw new Error(GENERIC_SERVICE_ERROR);
}
{{< /highlight >}}

As we are no longer dealing with a potential `AxiosError` body the error processor gets more than a little slimmer:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/helpers/service.ts" title="/src/helpers/service.ts" >}}*
{{< highlight typescript >}}
export const unwrapServiceError = (serviceMethod: string, error: any): Error => {
  return new Error(
    `${serviceMethod} error: ${error.errors ? error.errors[0] : error.message}`
  );
};
{{< /highlight >}}

## Result

Putting it all together I feel that it makes my service functions a bit more streamlined and easier to read:

file: *{{< newtabref href="https://github.com/code-chimp/vanilla-redux-template/blob/main/src/services/user/UsersApi.ts" title="/src/services/user/UsersApi.ts" >}}*
{{< highlight typescript >}}
import IUser from '../../@interfaces/IUser';
import { createGetRequest, processApiResponse, unwrapServiceError } from '../../helpers';

export const fetchUsers = async (): Promise<Array<IUser>> => {
  try {
    const response = await fetch(createGetRequest('/users'));

    return await processApiResponse<Array<IUser>>(response);
  } catch (e) {
    throw unwrapServiceError('UsersApi.fetchUsers', e);
  }
};
{{< /highlight >}}

As always feel free to drop me a line if you see anything you feel could be improved. Thank you for dropping by.
[Reference Project][van]

[van]: https://github.com/code-chimp/vanilla-redux-template 'My base template for new Redux projects'
[ax]: https://axios-http.com/ 'Promise based HTTP client for the browser and node.js'
[ftch]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API 'The Fetch API provides an interface for fetching resources'
[ng]: https://angular.io/ "Modern web developer's platform"
[node]: https://nodejs.org/en/ 'Open-source, cross-platform JavaScript runtime environment'
