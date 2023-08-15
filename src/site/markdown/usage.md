## Link Handler Usage


### Building links

The [Link Handler][link-handler] is a Sling Model and can be adapted either from a request or a resource. It automatically reads the context-specific configuration for the Site URLs of the [URL Handler][url-handler] based on the resource path of the current request or the path of the resource adapted from.

Example:

```java
LinkHandler linkHandler = request.adaptTo(LinkHandler.class);

// build link stored in current resource
Link link = linkHandler.get(resource).build();

// build link targeting the given content page with a different selector and extension
Link link = linkHandler.get(contentPage).selector("sel1").extension("pdf").build();

// check if link is valid and get markup
if (link.isValid()) {
  String markup = link.getMarkup();
  // ...
}
```

Alternatively you can inject the `LinkHandler` into your Sling Model using the `@Self` annotation if the model itself adapts from request or resource.

The link handler uses a "builder pattern" so you can flexibly combine the different link generation options.
See [LinkBuilder][link-builder] for all options.


### Link properties in resource

When storing a link in a resource multiple properties are used to describe the link. The properties depend on the link type implementations, these are the most important properties supported by the built-in link types:

* `linkType`: Type of links as chosen by the editor (e.g. internal, external or media)
* `linkContentRef`: Path of internal content page to link to
* `linkMediaRef`: Path of media asset (e.g. DAM asset) to link to
* `linkExternalRef`: External URL to link to
* `linkWindowTarget`: Target for window to open link in (e.g. "\_blank")
* `linkFragment`: Fragment part to add to link URL
* `linkQueryParam`: Query parameters to add to link URL

Further properties are defined in [LinkNameConstants][link-name-constants]. It is recommended to define an edit dialog that shows only the properties supported for the selected link type after choosing one.

If using the [Link Reference Container Granite UI component][graniteui-components] to show link related properties in the edit dialog be aware that some of the properties defined in the `LinkNameConstants` do not have a representation by default. Eg: it is required to add `linkFragment` and `linkQueryParam` separately in order to add fragment and query string support to the link generation.


### Using links in HTL/Sightly template

To resolve a link inside a HTL template you can use a generic Sling Model for calling the handler: [ResourceLink](apidocs/io/wcm/handler/link/ui/ResourceLink.html)

HTL template example:

```html
<sly data-sly-use.link="io.wcm.handler.link.ui.ResourceLink"/>
<a data-sly-attribute="${link.attributes}" data-sly-test="${link.valid}">
  ${properties.linkTitle}
</a>
<div class="cq-placeholder" data-emptytext="${component.title}" data-sly-test="${!link.valid}"></div>
```

In this case the anchor defined in the sightly template is used, but all attributes of the `a` element are overwritten with those returned by the link handler. This is primary the href attribute, but may contain further for defining the link target or custom metadata for user tracking.



### Configuring and tailoring the link resolving process

Optionally you can provide an OSGi service to specify in more detail the link resolving needs of your application. For this you have to extend the [LinkHandlerConfig][link-handler-config] class. Via [Context-Aware Services][sling-commons-caservices] you can make sure the SPI customization affects only resources (content pages, DAM assets) that are relevant for your application. Thus it is possible to provide different customizations for different applications running in the same AEM instance.

With this you can:

* Define which link types are supported by your application or include your own ones
* Define which markup builders are supported by your application or include your own ones
* Define custom pre- and postprocessors that are called before and after the link resolving takes place
* Implement a method which decides whether a content page is allowed to link to or not
* Implement a method which decides whether a content page is a redirect page or not

Example:

```java
@Component(service = LinkHandlerConfig.class, property = {
    ContextAwareService.PROPERTY_CONTEXT_PATH_PATTERN + "=^/content/(dam/)?myapp(/.*)?$"
})
public class LinkHandlerConfigImpl extends LinkHandlerConfig {

  private static final List<Class<? extends LinkType>> LINK_TYPES =
      ImmutableList.<Class<? extends LinkType>>of(
          InternalLinkType.class,
          InternalCrossScopeLinkType.class,
          ExternalLinkType.class,
          MediaLinkType.class
      );

  @Override
  public List<Class<? extends LinkType>> getLinkTypes() {
    return LINK_TYPES;
  }

  @Override
  public boolean isRedirect(Page page) {
    String template = page.getProperties().get(NameConstants.PN_TEMPLATE, String.class);
    return StringUtils.equals(template, "/apps/sample/templates/redirect");
  }

}
```

Schematic flow of link handling process:

1. Start link handler processing
2. Detect link type, store result in link request
3. Apply preprocessors on link request
4. Resolve link using link type, store result in link request
5. Generate markup using markup builder, store result in link request
6. Apply postprocessors on link request


### Migrate components to Link Handler

If you migrate existing components that did not use the wcm.io Link Handler before, it's likely you used only one single property which contained all types of links - internal, external, DAM asset. This storage as one single property is not compatible with the Link Handler, which uses at least two property (link type and link reference), and the second property depends on the link type.

But you can configure a special "fallback mode" on the component resource by setting a property - example:

```javascript
{
  "wcmio:linkTargetUrlFallbackProperty": "linkUrl"
}
```

When this property is set, the component is also able to read link target information from the property "linkUrl" and tries to auto-detect it's type. When the component is edited using the Link Handler link dialog tab, it's read from this property as well. Once the dialog is saved the property is cleared and the new property names as used by the Link Handler are used.



[link-handler]: apidocs/io/wcm/handler/link/LinkHandler.html
[link-builder]: apidocs/io/wcm/handler/link/LinkBuilder.html
[link-name-constants]: apidocs/io/wcm/handler/link/LinkNameConstants.html
[link-handler-config]: apidocs/io/wcm/handler/link/spi/LinkHandlerConfig.html
[url-handler]: ../url/
[sling-commons-caservices]: ../../sling/commons/context-aware-services.html
[graniteui-components]: graniteui-components.html
