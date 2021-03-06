# UI Extension examples

/!\ this is a feature preview that needs to be activated on your project beforehand

## About extensions

UI extensions allow you to replace components in the content editor by your own self-hosted app.
This app is displayed in an iframe and communicates with the content editor using a javascript SDK.

An extension is "just a webapp":

- you can use any language and framework to build it (eventually, as the current SDK version is for React)
- host it on your own domain
- build flows requiring server-side treatment, like authenticating with another 3rd party platform
- etc..

## Quick example: extending a form field (in React)

### Build your own field and render it on a page:

```jsx
import { useState } from "react";
import ReactDOM from "react-dom";

const MyField = () => {
  const [value, onChange] = useState(); // we want to sync this with GraphCMS later

  return <MyComplexInputComponent value={value} onChange={onChange} />;
};

ReactDOM.renderToString(<MyField />, "#app");
```

### Integrate the UI extensions SDK

```jsx
import { useState } from "react";
import ReactDOM from "react-dom";

import { Wrapper, useUiExtension } from "@grapcms/uix-react-sdk";
import type {
  ExtensionDeclaration,
  FieldExtensionType,
} from "@grapcms/uix-react-sdk";

const Extension = () => {
  // When displaying the extensions, GraphCMS will add a unique identifier 'extensionUid' to the url
  const queryParams = new URLSearchParams(window.location.search);
  const uid = queryParams.get("extensionUid");

  // We need a quick declaration of what this extension is about
  const declaration = {
    extensionType: "field", // we extend a form field
    fieldType: FieldExtensionType.STRING, // that handles string values
    name: "My own super field",
  };
  return (
    <Wrapper uid={uid} declaration={declaration}>
      <MyField />
    </Wrapper>
  );
};

const MyField = () => {
  // const [value, onChange] = useState();
  const { value, onChange } = useUiExtension(); // and we use the sdk hook to connect state

  return <MyComplexInputComponent value={value} onChange={onChange} />;
};

ReactDOM.renderToString(<MyField />, "#app");
```

And that's it !

## Adding the extension to GraphCMS:

/!\ this is a feature preview that needs to be activated on your project beforehand

### Register it in your app settings

### Update the renderer of one of your fields:

In the api playground, select the management api.

```graphql

mutation useExtensions($fieldId: ID!, $rendererConfig: JSON!) {
  updateSimpleField(data: {id: $fieldId, formConfig: {renderer: "CUSTOM", config: $rendererConfig}}){
    migration {
      id
    }
  }
}

Params:
{
  "fieldId": "qwertyuiop1234567890", // the ID of the field you want to replace
  "rendererConfig": { "extensionId": "poiuytrewq0987654321" } // the ID of the UI extension you just registered
}

```

## Type definition for useUiExtension:

```ts
export interface ExtensionType {
  // name of the field in the form (may differ from field apiId)
  name: string;
  // current locale on localized field
  locale?: string;

  // field state sync
  value: any;
  onChange: <T extends HTMLElement = HTMLElement>(event: ChangeEvent<T> | any) => Promise<void>;
  onFocus: <T extends HTMLElement = HTMLElement>(event?: FocusEvent<T>) => Promise<void>;
  onBlur: <T extends HTMLElement = HTMLElement>(event?: FocusEvent<T>) => Promise<void>;
  meta?: {  // @see https://final-form.org/docs/react-final-form/types/FieldRenderProps
    active: boolean;
    error: any;
    touched: boolean;
  };

  // fullscreen mode
  isExpanded?: boolean;
  expandField?: (expand: boolean) => unknown;

  // open the asset picker (see the Asset type in your Content Api)
  openAssetPicker: () => Promise<null | Asset>;

  // access to the form state and other form fields
  form?: {
    change: <Value = any>(name: string, value: Value) => Promise<void>;
    getState: <Values = Record<string, any>>() => Promise<FormState<Values>>; // see https://final-form.org/docs/final-form/types/FormState
    getFieldState: <Value = any>(fieldName: string) => Promise<FieldState<Value>>; // see https://final-form.org/docs/final-form/types/FieldState

  // details about the field you are currently extending
  field?: {
    id: string;
    apiId: string;
    description: string | null;
    displayName: string;
    isList: boolean;
    isLocalized?: boolean;
    isRequired?: boolean;
    isUnique?: boolean;
    isPreview: boolean; // true when in version or stage comparison view
    type: FieldExtensionType;
    model: {
      apiId: string;
      apiIdPlural: string;
      id: string;
      description: string | null;
      displayName: string;
      isLocalized: boolean;
    };
  };

  // information needed to be able to use the project api's
  context?: {
    project: {
      id: string;
      name: string;

      // management api access
      mgmtApi: string;
      mgmtToken: string;
    };
    environment: {
      id: string;
      name: string;

      // content api access
      endpoint: string;
      authToken: string;
    };
  };

  // extension status
  extension: {
    onReady: () => void;
    config: Record<string, any>;
    status: "connected" | "connecting" | "error" | "disconnected";
  };
}
```
