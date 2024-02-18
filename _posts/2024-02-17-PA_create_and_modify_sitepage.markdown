---
layout: post
title:  "Power Automate and SharePoint Online - How to create a Site Page from a Template and update it's content with Power Automate only"
date:   2024-02-17 23:53:00 +0100
categories: SPO PowerAutomate Hacky
---



# How to create a Site Page from a Template and update it's content with Power Automate only

Have you ever thought it would be nice if you could create News Posts (or any other Site Page) in SharePoint online using Power Automate?

Well .. I didn't. But turns out I needed to, and it was a fun little project.

So here we go:

## Outline and Setup

A Document library containing pictures of all employees is used to trigger a Power Automate Flow which creates a News Post welcoming new Employees in the company and presenting them to their colleagues.

The pictures have Metadata like Employee Name, Department etc.

The Layout of the News Post is defined in a Template containing Placeholders where the Employees Name, Department and Picture should go.

I'm not going to talk much about the Document Library itself, since it is only one of many possible sources in such a scenario. The main focus is on the Flow.

## Implementation 


### The template
First thing we need is a Template Page. (This is not actually a template, it's just the Page we copy every time our Flow runs)

I built my Page layout and put in the static elements like Banner image and static text.
Then I used Placeholders between `<>` to replace them later with the real data.
For the image I uploaded a Sample image to the document library which later will contain all the pictures for these Posts and added it to the Page by link. This way, I only need to replace the filename of the Image within the link. (The path of the library remains unchanged)
![alt text](2024-02-17-PA_create_and_modify_sitepage/image-7.png)

I saved my page as `template-welcome.aspx`.

### Copying the template
Now for the somewhat tricky part.

Whenever the flow is run, we need to get the content of `template-welcome.aspx`, replace the placeholders with real data and store this new content in a new aspx file.

The first step in our flow will be to create a copy of the template with a decent name.
This can be done using `Send an HTTP request to SharePoint` and the following parameters:

```
POST

_api/web/getfilebyserverrelativeurl('/sites/<your site>/SitePages/<Filename of the template>')/copyto(strnewurl='/sites/<your site>/SitePages/<Filename of the new page>.aspx',boverwrite=false)

Headers:
content-type: application/json;odata=verbose
accept: application/json;odata=verbose

Body: empty
```

![alt text](2024-02-17-PA_create_and_modify_sitepage/image.png)


As you see in the Image above, I used variables to store the name of the template and Compose, to dynamically generate the filename. Using `concat` to create a string out of the pictures Id in the document library and the Employees Name I'm confident to avoid naming conflicts.


### Getting the page content
Now, to get the content of the existing file, we can use a trick I learned from a post on [powerusers.microsoft.com](https://powerusers.microsoft.com/t5/Building-Flows/Change-the-content-of-a-SharePoint-page/td-p/1558883) by [SilannaIT-Dale](https://powerusers.microsoft.com/t5/user/viewprofilepage/user-id/636648)



The SharePoint API exposes the Method `CheckOutPage` which ... checks out the page ... but more importantly: Returns the pages content after doing so.

With the following HTTP request to the SharePoint API we can get the content of the newly created copy of our template.

```
POST 
_api/sitepages/pages(<ID of the Page>)/CheckOutPage

Headers:
content-type: application/json;odata=verbose
accept: application/json;odata=verbose

Body: empty
```

![alt text](2024-02-17-PA_create_and_modify_sitepage/image-1.png)

![alt text](2024-02-17-PA_create_and_modify_sitepage/image-2.png)


### Modifying the Page content
Having the content of our Page, we now need to replace the placeholders with actual values.



[!NOTE]
My first attempt to use Parse JSON on the body returned by the CheckOutPage Method was a total bust because i ran into some very strange behavior.
While the output of the Parse JSON step was perfectly fine, the dynamic content it produced was always empty. After manually modifying the Schema, validating the schema and even using PowerShell to parse the json in the hope of finding some oversight in the schema i finally gave up.
That's why in the following Screenshots you'll find all references to the content of the page as expressions.



While the CheckOutPage Method returned a lot in the body, we actually don't need all of it. 


What we want to change is the `CanvasContent1` property. It contains the actual content of the Page.
Here we find the Text I wrote in the Template as well as the Picture and it's src attribute where we can replace the filename to point to a different image.

First, We initialize a variable with the content of `CanvasContent1`

![alt text](2024-02-17-PA_create_and_modify_sitepage/image-3.png)

Next we can use the replace expression to replace the placeholders with the actual values.
In this screenshot you see how I replace the string "Sample.png" with the FilenameWithExtension Attribute I got from the trigger. (It contains the Filename + Extension of the file that was selected when the Flow was triggered)
I actually used multiple compose steps in row and only replaced one Placeholder in every step.
It would work to nest the replace statements but I find it easier to read this way.
PS: What we can't do is self-reference a Variable. So if you plan on using Set Variable instead of compose you're going to be disappointed.

### Saving the changes

![alt text](2024-02-17-PA_create_and_modify_sitepage/image-4.png)

In this screenshot you see how everything got put together to create the body for the last request.

![alt text](2024-02-17-PA_create_and_modify_sitepage/image-5.png)

If you're asking how I chose those attributes of all that where returned by the CheckOutPage Method the answer is surprisingly simple.

1. These are the values I need/want
2. Using the developer tools in the browser while saving a Page as draft, I could look at how the SharePoint UI made the requests containing exactly those attributes. (except for the description, which I wanted to have in there)  



The last step is another HTTP request to the API calling the Method `savepageasdraft` and delivering the new page content as body:


```
POST
_api/sitepages/pages(<ID of the Page>)/savepageasdraft

Headers:
content-type: application/json;odata=verbose
accept: application/json;odata=verbose

Body: {
  "__metadata": {
    "type": "SP.Publishing.SitePage"
  },
  "LayoutWebpartsContent": "@{variables('LayoutWebpartsContent')}",
  "CanvasContent1": "@{outputs('Compose_-_Replace_Image_URL_in_Canvas_Content')}",
  "AuthorByline": [
    "i:0#.f|membership|username@contoso.com"
  ],
  "TopicHeader": null,
  "BannerImageUrl": "@{variables('BannerImageUrl')}",
  "Description": "My new description",
  "Title": "@{outputs('Compose_-_Replace_Department_Name_in_Title')}"
}
```

![alt text](2024-02-17-PA_create_and_modify_sitepage/image-6.png)

In this particular case i wanted the user to be able to look at the Post and maybe add some custom Text before publishing it. 
If you want to publish the page directly from your Flow you can use the `publish` Method of the API:

```
POST 
_api/sitepages/pages(<ID of the Page>)/publish
```

