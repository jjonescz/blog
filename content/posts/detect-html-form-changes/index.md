---
title: Detecting changes of HTML forms
slug: detect-html-form-changes
date: 2021-03-07T17:02:19Z
summary: How to automatically detect form changes and prevent users from losing
  entered data or submitting empty forms.
tags:
  - HTML
  - TypeScript
  - JavaScript
  - jQuery
categories:
  - Programming
projects:
  - KnowledgePicker
cover:
  image: cover.png
  alt: Prevent user from losing changes in a form
  relative: true
ShowToc: true
TocOpen: false
---

In [KnowledgePicker](https://knowledgepicker.com), we have several different kinds of HTML forms.

- Some are simple and they don't require any special `onsubmit` handling, e.g., [login](https://knowledgepicker.com/login) form.
- Others require users to enter non-trivial amount of data and we should ensure these data are **not unintentionally lost**.
  This is for example [edit profile](https://knowledgepicker.com/profile) form where users can write their bio and enter many external links (which might take many minutes to think of and gather).
- Finally there are forms that should **not be submitted without changes**.
  This might sound strange at first but there is a good reason for this kind of forms.
  It's for example [edit resource](https://knowledgepicker.com/r/add) form&mdash;the form is first pre-filled with current resource data and it can be submitted (which will create new Edit Request) only if there are actually some changes.

  Note that this is not the same as [form validation](https://developer.mozilla.org/en-US/docs/Learn/Forms/Form_validation), because all required fields can be filled out but sending the form would still create an empty draft (because there are no *changes* since the last version) which we want to forbid.

I will introduce here a small TypeScript class that can handle the latter two cases automatically.
In the first case, it will display the usual "Changes you made may not be saved. Leave? / Cancel?" dialog.
In the second case, it will display a dialog saying that you are trying to submit an empty draft which is not allowed (of course this should be also checked on server side).

{{< figure src="leave-site.png" class="screenshot"
    link="leave-site.png" rel="true"
    alt="Leave site? dialog" >}}

## Getting started

I used minimal amount of [jQuery](https://jquery.com/) because we use it in other parts of the project.
However, in this case it shouldn't be necessary and replacing it with vanilla JavaScript should be easy.

```ts
import $ from 'jquery';
```

We call form with changes a *dirty form*.
And therefore our class is called...

```ts
/** Tracks changes and warns user if he is trying to leave an empty form. */
export class DirtyFormDetector {
  dirty = false;
  form: JQuery;

  /**
   * @param form - The form whose fields will be checked for changes.
   */
  constructor(form: JQuery) {
    this.form = form;

    // Detect changes at startup. For `pageshow`, see
    // https://stackoverflow.com/a/8861236.
    $(window)
        .on('load', () => this.detectChanges())
        .on('pageshow', () => this.detectChanges());
    this.detectChanges();
  }
}
```

As you can see, this class is mainly focused on detecting form changes (probably the most common scenario).
Detecting empty form submission will be added later (if you need it).

## Detecting dirtiness

Let's add method that will detect whether form is changed or not.
It is originally inspired by a great [StackOverflow answer](https://stackoverflow.com/a/155812), however I also modified it quite a bit over time.

```ts
/** Determines whether form elements of `parent` (including the `parent` element
  * itself if appropriate) have been changed.
  *
  * @remarks Inspired by https://stackoverflow.com/a/155812.
  */
private static isDirty(parent: JQuery) {
  // Method `addBack` ensures that `parent` is also included in the results if
  // it passes the filter.
  for (const element of parent.find('input').addBack('input')) {
    // If element has no name, it won't be submitted => ignore it.
    if ($(element).attr('name') === undefined) continue;

    const type = element.type;
    if (type == 'checkbox' || type == 'radio') {
      if (element.checked != element.defaultChecked) {
        return true;
      }
    } else {
      if (!strEqual(element.value, element.defaultValue)) {
        return true;
      }
    }
  }
  for (const element of parent.find('textarea').addBack('textarea')) {
    // If element has no name, it won't be submitted => ignore it.
    if ($(element).attr('name') === undefined) continue;

    if (!strEqual(element.value, element.defaultValue)) {
      return true;
    }
  }
  for (const element of parent.find('select').addBack('select')) {
    // If element has no name, it won't be submitted => ignore it.
    if ($(element).attr('name') === undefined) continue;

    const type = element.type;
    if (type == 'select-one' || type == 'select-multiple') {
      for (const option of element.options) {
        if (option.selected != option.defaultSelected) {
          return true;
        }
      }
    }
  }
  return false;

  /** Normalizes line endings in `s`. */
  function normalizeNewLines(s: string) {
    return s.replace(/(\r\n|\r)/g, '\n');
  }

  /**
    * Compares two strings for equality regardless characters used for line
    * endings.
    */
  function strEqual(x: string, y: string) {
    return normalizeNewLines(x) == normalizeNewLines(y);
  }
}
```

Note that we have to separately consider `input`, `select` and `textarea` form fields.
Also note that `input[type=hidden]` fields will be always considered as not changed which is probably OK but you can change method `isDirty` to include custom logic for this case if you need to.

## Displaying dialog

Now let's get to the action.

```ts
/** String prefixed to page's title if there are unsaved changes. */
const prefix = '* ';

/**
  * Enables warning about form dirtiness before page unload and adds marker to
  * page's title that it's dirty.
  */
private setDirty() {
  window.onbeforeunload = (e: BeforeUnloadEvent) => {
    // If something's changed, warn the user.
    e.preventDefault();

    // Legacy mode: some browsers require a string message to be returned
    // (either as `e.returnValue` or using `return`). See
    // https://developer.mozilla.org/en-US/docs/Web/API/Window/beforeunload_event.
    const message =
        'You have unsaved changes that will be lost if you leave this page.';
    e.returnValue = message;
    return message;
  };
  document.title = prefix + document.title;
}

/** Undoes effects of `setDirty`. */
private clearDirty() {
  window.onbeforeunload = null;
  document.title = document.title.slice(prefix.length);
}
```

## Putting it together

And finally let's add a method that will be called to detect changes in a specific element or the whole form, as needed.

```ts
/** Auto-detects form's dirtiness. */
detectChanges(element?: JQuery) {
  // Check either all elements in the form or just the provided element.
  let newDirty = false;
  if (element !== undefined) newDirty = DirtyFormDetector.isDirty(element);

  // If only one element was checked and it was not dirty, we must also check
  // the whole form, because other elements could be dirty.
  if (!newDirty) newDirty = DirtyFormDetector.isDirty(this.form);

  this.dirty = newDirty;
}
```

Remember, this one is called on page load (we did that in `constructor` of `DirtyFormDetector`), but we should also call it whenever some `input` of the given `form` changes.
I do it manually as shown below (because every form is a bit different) but I am sure you can easily integrate it into the `DirtyFormDetector` class.

And here is how you actually use our new `DirtyFormDetector` class:

```ts
import { DirtyFormDetector } from './dirty-form-detector';
import $ from 'jquery';

// Enable dirty form module on my form.
const form = $('#myForm');
const dirtyForm = new DirtyFormDetector(form);
form.on('input', 'input, textarea, select', e =>
  dirtyForm.detectChanges($(e.currentTarget))
);
form.on('submit', () => {
  // If the form is going to be submitted, disable warning that it's dirty.
  dirtyForm.dirty = false;
});
```

### Detecting empty drafts

If you also want to have this &#8220;weird&#8221; form which shouldn't be submitted unchanged, here's what to instead.

```ts
form.on('submit', () => {
  // Warn the user if he's trying to submit unchanged form.
  if (!dirtyForm.dirty) {
      e.preventDefault();
      $('#unchangedAlert').modal('show'); // Bootstrap modal
  } else {
    // If the form is going to be submitted, disable warning that it's dirty.
    dirtyForm.dirty = false;
  }
});
```

Note that I use [Bootstrap modal](https://getbootstrap.com/docs/4.6/components/modal/) to show the "You are trying to submit unchanged form" warning.
