## Property KB: placeholder dynamics

Let's clarify how placeholder may evolve.

Placeholders may be of three types types:

### Standard textual

The label and description are given by us and it's unchangeable, the value is changeable.

![standard textual](./images/standard%20textual.png)

When user starts modifying it, it turns red to show modification occurred. It stays like this until saving.

![standard textual modified](./images/standard%20textual%20modified.png)

> Note: the arrow on the right retrieves the last saved value.


### Standard media 

The label and description are given by us and it's unchangeable, the value can is changeable

![standard media](./images/standard%20media.png)

When clicking camera or video icon, opens file system. The other icons save or delete the media.

> IMPORTANT: When media is uploaded, the field goes in modified mode, that is, with red border
> ![standard media uploaded](./images/standard%20media%20uploaded.png)
> If the file is heavier than 25Mb, can the value turn red as well?

### Custom textual or media (based on the user interaction). 

Appears when user clocks on **Add a placeholder**. The user can customize:
    - label, 
    - description
    - value, and the value can be either textual or media based on the user input

![custom placeholder not changed](./images/custom%20placeholder%20not%20changed.png)

> Note ON LABEL: when user clicks on the modify icon next to the label, it shows a dropdown that looks in all custom labels already set (this, to avoid doubled custom placeholders)
> ![custom label search](./images/custom%20label%20search.png)

> Note ON DESCRIPTION: when user clicks on the modify icon next to the description, the description becomes editable
> ![editable description](./images/custom%20description.png)

> IF USER ENTERS TEXT VALUE: when a value is set, the appearance is the same as the one of standard textual placeholders when modified

> IF USER ENTERS MEDIA VALUE: when a value is set, the appearance is the same as the one of standard media placeholders when modifief


## Save buttons
- 1 at the box level
- 1 at the page level

> IMPORTANT: see modals in the Figma document