---
title: Building UI Plugins Using Custom Components
description: A guide to building NativeScript user interface plugins, including how to bootstrap your plugins, and how to structure your native code.
position: 35
slug: building-ui-plugins-custom-components
---

# Building UI Plugins using Custom Components

Whenever needed UI can be shown by a plugin just by exposing a custom component, e.g. some platform-specific functionality that renders UI itself. To demonstrate that, this article explains how to create a simple button plugin.


## Prerequisites

The article contains information applicable to apps built with NativeScript 3.x.x or newer version

## Bootstrap Your Plugin 

First things first - you start off from a regular plugin. You can check the [Building Plugins article]({%slug building-plugins%}) for reference.

## Common Code

Let's say you want to build a simple button which you can use like:

```XML
    <ui:MyButton text="MyButton1" tap="onTap" />
```

This can be accomplished by wrapping the platform-specific buttons (iOS's UIButton and Android's android.widget.Button) and expose it from a common MyButton class.   

You can implement this by creating four files:

- **my-button.d.ts** - holds the declarations of MyButton class, its properties "text" and "myOpacity", enables auto-complete in some IDEs. 
- **my-button.common.ts** - contains the logic accessible from the apps.
- **my-button.ios.ts** - holds the iOS-specific logic for creation of the native view (UIButton)
- **my-button.android.ts** - holds the Android-specific logic for creation of the native view (android.widget.Button)

This file holds type definitions for the common logic that will be imported in the app that is using the plugin.
_my-button.d.ts_
```TypeScript
import { View, Style, Property, CssProperty, EventData } from "@nativescript/core";

export class MyButton extends View {
    // static field used from component-builder module to find events on controls.
    static tapEvent: string; 

    // Defines the text property.
    text: string;

    // Overload 'on' method so that it provides intellisense for 'tap' event.
    on(event: "tap", callback: (args: EventData) => void, thisArg?: any);

    // Needed when 'on' method is overriden.
    on(eventNames: string, callback: (data: EventData) => void, thisArg?: any);
}

export const textProperty: Property<MyButton, string>;
export const myOpacityProperty: CssProperty<Style, number>;
```

In the following way you create the common logic:
_my-button.common.ts_
```TypeScript
import { MyButton as ButtonDefinition } from "./my-button";
import { View, Style, Property, CssProperty, isIOS } from "@nativescript/core";

export const textProperty = new Property<MyButtonBase, string>({ name: "text", defaultValue: "", affectsLayout: isIOS });

// using myOpacity instead of opacity as it will override the one defined in `@nativescript/core`
export const myOpacityProperty = new CssProperty<Style, number>({
    name: "myOpacity", cssName: "my-opacity", defaultValue: 1, valueConverter: (v) => {
        const x = parseFloat(v);
        if (x < 0 || x > 1) {
            throw new Error(`opacity accepts values in the range [0, 1]. Value: ${v}`);
        }

        return x;
    }
});
export abstract class MyButtonBase extends View implements ButtonDefinition {
    public static tapEvent = "tap";
    text: string;

    // Exposing myOpacity style property through MyButton.
    // This is all optional. If not exposed users will have to set it
    // through style: <control:MyButton style.myOpacity='0.4' />.
    get myOpacity(): number {
        return this.style.myOpacity;
    }
    set myOpacity(value: number) {
        this.style.myOpacity = value;
    }
}

// Augmenting Style definition so it includes our myOpacity property
declare module "@nativescript/core/ui/styling/style" {
    interface Style {
        myOpacity: number;
    }
}

// Defines 'text' property on MyButtonBase class.
textProperty.register(MyButtonBase);

// Defines 'myOpacity' property on Style class.
myOpacityProperty.register(Style);
 
// If set to true - nativeView will be kept in memory and reused when some other instance 
// of type MyButtonBase needs nativeView. Set to true only if you are sure that you can reset the
// nativeView to its initial state. When true will improve application performance. 
MyButtonBase.prototype.recycleNativeView = false; 
```

You see "text" and "myOpacity" properties are defined in this file and also recycleNativeView is set to "false". To read more how these declarations work refer the [Properties article]({%slug properties%}).

## Platform-specific Code

Writing the platform-specific implementations, the following overrides need to be considered:
- `createNativeView` - you override this method, create and return your nativeView 
- `initNativeView` - in this method you setup listeners/handlers to the nativeView 
- `disposeNativeView` - in this method you clear the reference between nativeView and javascript object to avoid memory leaks as well as reset the native view to its initial state if you want to reuse that native view later.

_my-button.android.ts_
```TypeScript
import { MyButtonBase, textProperty, myOpacityProperty } from "./my-button.common";

let clickListener: android.view.View.OnClickListener;

// NOTE: ClickListenerImpl is in function instead of directly in the module because we 
// want this file to be compatible with V8 snapshot. When V8 snapshot is created
// JS is loaded into memory, compiled & saved as binary file which is later loaded by
// Android runtime. Thus when snapshot is created we don't have Android runtime and
// we don't have access to native types.
function initializeClickListener(): void {
    // Define ClickListener class only once.
    if (clickListener) {
        return;
    }

    // Interfaces decorator with implemented interfaces on this class
    @Interfaces([android.view.View.OnClickListener])
    class ClickListener extends java.lang.Object implements android.view.View.OnClickListener {
        public owner: MyButton;

        constructor() {
            super();
            // Required by Android runtime when native class is extended through TypeScript.
            return global.__native(this);
        }

        public onClick(v: android.view.View): void {
            // When native button is clicked we raise 'tap' event.
            const owner = (<any>v).owner;
            if (owner) {
                owner.notify({ eventName: MyButtonBase.tapEvent, object: owner });
            }
        }
    }

    clickListener = new ClickListener();
}

export class MyButton extends MyButtonBase {

    // added for TypeScript intellisense.
    nativeView: android.widget.Button;

    /**
     * Creates new native button.
     */
    public createNativeView(): Object {
        // Initialize ClickListener.
        initializeClickListener();

        // Create new instance of android.widget.Button.
        const button = new android.widget.Button(this._context);

        // set onClickListener on the nativeView.
        button.setOnClickListener(clickListener);

        return button;
    }

    /**
     * Initializes properties/listeners of the native view.
     */
    initNativeView(): void {
        // Attach the owner to nativeView.
        // When nativeView is tapped we get the owning JS object through this field.
        (<any>this.nativeView).owner = this;
        super.initNativeView();
    }

    /**
     * Clean up references to the native view and resets nativeView to its original state.
     * If you have changed nativeView in some other way except through setNative callbacks
     * you have a chance here to revert it back to its original state 
     * so that it could be reused later.
     */
    disposeNativeView(): void {
        // Remove reference from native view to this instance.
        (<any>this.nativeView).owner = null;

        // If you want to recycle nativeView and have modified the nativeView 
        // without using Property or CssProperty (e.g. outside our property system - 'setNative' callbacks)
        // you have to reset it to its initial state here.
        super.disposeNativeView();
    }

    // transfer JS text value to nativeView.
    [textProperty.setNative](value: string) {
        this.nativeView.setText(value);
    }

    // gets the default native value for opacity property.
    // Alpha could be controlled from Android theme.
    // Thus we take the default native value from the nativeView.
    // If view is recycled the value returned from this method
    // will be passed to [myOpacityProperty.setNative]
    [myOpacityProperty.getDefault](): number {
        return this.nativeView.getAlpha()
    }

    // set opacity to the native view.
    [myOpacityProperty.setNative](value: number) {
        return this.nativeView.setAlpha(value);
    }
}
```

> **NOTE**: In Android, avoid access to native types in the root of the module (note that ClickListener is declared and implemented in a function which is called at runtime). This is specific for the [V8 snapshot feature](https://www.nativescript.org/blog/improving-app-startup-time-on-android-with-webpack-v8-heap-snapshot) which is generated on a host machine where android runtime is not running. What is important is that if you access native types, methods, fields, namespaces, etc. at the root of your module (e.g. not in a function) your code won't be compatible with V8 snapshot feature. The easiest workaround is to wrap it in a function like in the above `initializeClickListener` function.
 
_my-button.ios.ts_
```TypeScript
import { MyButtonBase, textProperty, myOpacityProperty } from "./my-button.common";

// class that handles all native 'tap' callbacks
@NativeClass()
class TapHandler extends NSObject {

    public tap(nativeButton: UIButton, nativeEvent: _UIEvent) {
        // Gets the owner from the nativeView.
        const owner: MyButton = (<any>nativeButton).owner;
        if (owner) {
            owner.notify({ eventName: MyButtonBase.tapEvent, object: owner });
        }
    }

    public static ObjCExposedMethods = {
        "tap": { returns: interop.types.void, params: [interop.types.id, interop.types.id] }
    };
}

const handler = TapHandler.new();

export class MyButton extends MyButtonBase {

    // added for TypeScript intellisense.
    nativeView: UIButton;

    /**
     * Creates new native button.
     */
    public createNativeView(): Object {
        // Create new instance
        const button = UIButton.buttonWithType(UIButtonType.System);

        // Set the handler as callback function.
        button.addTargetActionForControlEvents(handler, "tap", UIControlEvents.TouchUpInside);

        return button;
    }

    /**
     * Initializes properties/listeners of the native view.
     */
    initNativeView(): void {
        // Attach the owner to nativeView.
        // When nativeView is tapped we get the owning JS object through this field.
        (<any>this.nativeView).owner = this;
        super.initNativeView();
    }

    /**
     * Clean up references to the native view and resets nativeView to its original state.
     * If you have changed nativeView in some other way except through setNative callbacks
     * you have a chance here to revert it back to its original state 
     * so that it could be reused later.
     */
    disposeNativeView(): void {
        // Remove reference from native listener to this instance.
        (<any>this.nativeView).owner = null;
        
        // If you want to recycle nativeView and have modified the nativeView 
        // without using Property or CssProperty (e.g. outside our property system - 'setNative' callbacks)
        // you have to reset it to its initial state here.
        super.disposeNativeView();
    }

    // transfer JS text value to nativeView.
    [textProperty.setNative](value: string) {
        this.nativeView.setTitleForState(value, UIControlState.Normal);
    }

    // gets the default native value for opacity property.
    // If view is recycled the value returned from this method
    // will be passed to [myOpacityProperty.setNative]
    [myOpacityProperty.getDefault](): number {
        return this.nativeView.alpha;
    }

    // set opacity to the native view.
    [myOpacityProperty.setNative](value: number) {
        return this.nativeView.alpha = value;
    }
}
``` 

In the above mentioned implementations we use singleton listener (for Android - `clickListener`) and handler (for iOS - `handler`) in order to reduce the need to instantiate native classes and to reduce memory usage. If possible it is recommended to use such techniques to reduce native calls.

For more details and the full source code of the described MyButton sample, check the [NativeScript UI Plugin (Custom button component) repo](https://github.com/NativeScript/nativescript-ui-plugin-custom). 


## Make Your Plugin Angular-Compatible

Having your UI plugin developed successfully you could easily make it Angular-compatible following the steps described in [Supporting Angular in UI Plugins article]({%slug supporting-angular-in-ui-plugins%}).
