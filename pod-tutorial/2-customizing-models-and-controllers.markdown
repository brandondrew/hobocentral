# Part 2 -- Customizing the App

In a short space of time, you have created a working app. Some parts of the generic app might
already be quite close to what you want, other parts will not. Now it is time to begin customizing
the app to your needs. 

Your initial inclination might be to dive in to `app/views` and start modifying the individual 
page views. However, you will soon find that initially there are very few files for the individual 
pages in the generic app. With the exception of the front page, Hobo doesn't generate the generic views in your app. They are being rendered using a set of generic page tags provided by the Hobo Rapid library. DRYML allows these page tags to be customized in powerful ways without having to re-define parts of the page that you don't want to change. 

But before we dive into DRYML we can change the behavior of the generated UI in several ways by making changes to our models and controllers, and these are a good place to start.

1. [Model Layer](#model-layer)
2. [Resource Controllers](#resource-controllers)

---

## <a name="model-layer">Model Layer</a>

### Adding permissions

Hobo's model level permissions are a good place to start when customizing your app. Although
they are designed to give you model level integrity they are also used by many of Hobo Rapid's
tags. This means that the generic views automatically change based on the permissions of the
current user.

### The administrator role

By default, permissions for categories are defined like this:

File: app/models/category.rb

    # --- Hobo Permissions --- #

    def creatable_by?(user)
      user.administrator?
    end

    def updatable_by?(user, new)
      user.administrator?
    end

    def deletable_by?(user)
      user.administrator?
    end

    def viewable_by?(user, field)
      true
    end
{: .ruby}

What's happening here? Whenever Hobo needs to know if a user is allowed to make a change, it calls one of these methods, passing the currently logged in user. If the method returns true, the operation is allowed, otherwise it is not. By returning `user.administrator?`, we're effectively saying that the administrator can perform this operation, but no-one else can.

By default, Hobo assigns the first user to sign up as administrator. If you haven't already signed up a user, do so now.

When logged in as the administrator you will see that the categories section has a link to create a new category:

<img src="/images/tutorial/new-category-link.png">

Follow that link and you'll see Hobo has automatically created a form for the category:

<img src="/images/tutorial/new-category-form.png">

You can now create a category. By default Hobo takes you to the 'show' page for the new category. You can use the navigation bar to get back to the "Categories" index page to create more categories.

You should see that you are now able to rename, and delete categories as well.

If you log out and then sign up as another user, that user will not be automatically signed up as an administrator. The new category link will not be present and the new category form will not be available. 

It is worth noting that although Hobo introduces the concept of an administrator you are free to ignore it or change it in any way to suite your needs. You can redefine `administrator?` in your user model to anything you like, replace it with a role-based permission system, or remove it entirely.

### Allowing logged in users to create and update a model

As described above the default permissions allow anyone to view a particular model and only an administrator to create, update and delete. In our Pod app, we would like any registered user to be able to create, update and delete their own adverts. To do this, we begin by modifying the advert model permissions as follows:

File: app/models/advert.rb

    # --- Hobo Permissions --- #

    def creatable_by?(user)
      !user.guest?
    end

    def updatable_by?(user, new)
      !user.guest?
    end

    def deletable_by?(user)
      !user.guest?
    end

    def viewable_by?(user, field)
      true
    end
{: .ruby}

What we're saying here is that anyone who is logged in is allowed to create, change and delete any advert. That's far too permissive but it's a start. Try logging in as someone other than admin and creating some adverts.

You probably noticed that you can set the user who owns the advert to anyone. That's not what we want. The current user should be automatically assigned as the owner of the advert, and no one (except maybe admin) should be able to change that later.

All of this is easily handled by Hobo's permission system. First of all we need to tell Hobo that the `user` association on an advert, is what Hobo calls the 'creator association'. This means that Hobo will automatically set the user to be the currently logged in user when the advert is created. We just need to change the `belongs_to :user` declaration on the advert to:

File: app/models/advert.rb

    belongs_to :user, :creator => true
{: .ruby}
    
Next we need to alter the permissions so that the user is not allowed to manually set the user. This requires a bit more understanding of the permission system. We'll take the create and update stages in turn.

The method `creatable_by?` is called when Hobo needs to know if the current user is allowed to create a particular record. Hobo instantiates the object, in our case the new advert, but does not save it in the database. The fields are then set to the proposed values, and the `creatable_by?` method is called. So, inside `creatable_by?`, `self` is in the proposed state. Hobo is effectively asking the object to test itself: "are you in a legal state for this user to create you?".

Remember the `user` association has been automatically set, so the test can be written like this:

File: app/models/advert.rb

    def creatable_by?(user)
      self.user == user
    end
{: .ruby}
    
To paraphrase, we're stating that the currently logged in user (the `user` parameter) can create this advert, as long as they are the same user as specified in the `self.user` association. In other words, you can only create your own adverts. We could go further and write:

    def creatable_by?(user)
      self.user == user || user.administrator?
    end
{: .ruby}
    
Which would exempt the administrator from this restriction, allowing them to create adverts that belong to any user.

Make the change to `creatable_by?` and refresh the browser. You should see that the new advert form no longer contains the user selector. Hobo has discovered that you do not have permission to change that association, and has removed it.

If you create an advert and then navigate to its edit page, you'll see that the same restriction is not in place -- it is possible to change the advert's user. This is because create permission and update permission are defined separately. We need to modify `updatable_by?`.

`updatable_by?` works slightly differently to the other permission methods. When an update to a record is attempted (e.g. by a form submission), Hobo retrieves the record from the database, and creates a *duplicate* of it. The changes are made to the duplicate. The `updatable_by?` method is then called on the original *unchanged* record, passing the changed record as the `new` parameter. So when we implement `updateable_by?` on our advert model, we know that `self` is the advert as it now exists in the database, `user` is the user attempting to make the change, and `new` is the proposed new state of the advert. This gives us all the information we need to propose pretty much any security restriction.

There are two restrictions we need in this case: a user should only be able to modify their own adverts, and the `user` association itself should never be changed. Hobo provides a convenience method `same_fields?` for asserting that a field (or several fields) has not changed its value. The definition we want is:

    def updatable_by?(user, new)
      user == self.user && same_fields?(new, :user)
    end
{: .ruby}
    
Or, if we want to allow administrators to do whatever they like:

    def updatable_by?(user, new)
      (user == self.user && same_fields?(new, :user)) || user.administrator?
    end
{: .ruby}
    
If you make this change, you should see two changes in the UI: the edit advert form in the app no longer contains a user selector, and there is no edit link at all for adverts belonging to other users.

The app is looking pretty good now, despite the fact we haven't done any work at all in the view layer. We're not quite ready to move on yet though -- Hobo's automatic views have one or two more tricks up their sleeve.

## Annotating The Model

### Specifying the creator

By annotating an association we can specify the creator model. For example, "users" create "adverts".

In `app/models/advert.rb`, change `belongs_to :user` to

    belongs_to :user, :creator => true
{: .ruby}

This special association is picked up by Hobo's generic tags in several places. For example on an advert show page the creator is now displayed in the heading.

#### Before

<img src="/images/tutorial/creator_before.png">

#### After

<img src="/images/tutorial/creator_after.png">


### Model Hierarchy

**NOTE: These ideas are very experimental! They might change in a future Hobo release.**

Most domain models have some kind of innate hierarchy. Forums have topics, blog posts have comments, categories have adverts. There may be more than one level, e.g. forum topics each have their own collection of replies. If Hobo sees such a hierarchy in your models, the automatic interface will take it into account and you'll have an even better UI.

How does Hobo detect these relationships? They all have one thing in common - life-cycle dependencies. If you delete a blog-post, you're going to want the comments to be deleted too. If you delete a forum topic, you'll want all the replies deleted. In Active Record we declare these dependencies on our `has_many` relationships with `:dependent => :destroy` or `:dependent => :delete_all`. Hobo's automatic pages spot these declarations and adapt accordingly.

Aside: Does this feel too magical? Do we really want the automatic pages to be so clever? What if built in assumptions don't work in a particular situation? All this is really just about extending an already familiar concept - convention over configuration, a.k.a. sensible defaults. The interface you get out of the box is just a starting point. You can easily throw away some or all of the defaults, on a page by page basis or even site-wide. Hobo's goal is to get as close as possible to a good UI out of the box. Your job is then to take it the rest of the way and get the hand crafted UI your app needs. If you only have to change one or two things -- fantastic! If you end up changing everything, that's fine too.

In our app, there's really only one such relationship: adverts exist "inside" categories. We can declare this in the `Category` model by modifying the `has_many :adverts` declaration as follows:

File: app/models/category.rb

    has_many :adverts, :dependent => :destroy
{: .ruby}
    
That's it. You should see a couple of changes in the UI:

 * The show page for individual categories will now contain summary "cards" for each advert.
 * Each advert will provide a link back to its category, just above the title of the advert
 
It also makes sense that when a user is deleted from this app, their adverts should be removed too. Let's declare that rule in the user model:

File: app/models/user.rb

    has_many :adverts, :dependent => :destroy
{: .ruby}
    
If you click on one of the user links in the app, you should now see that all of a user's adverts are listed. Remember it's very easy to change this if the assumptions built in to the automatic pages don't work for your application -- these are just useful defaults.


We're now going to move on and look at customising the controllers. These automatic pages have some more smarts that respond to controller-level configuration, so we'll be able to improve the UI still more with just a few declarative changes.

----

## <a name="resource-controllers">Customizing controllers</a>

In this  chapter we'll make a few small controller changes and see how the automatic pages respond to give us a still more tailored UI.

### Removing an 'index' page

The front page and main nav bar decide which models to show based on the presence of an index page, or, more accurately an index *route*. For example, "Adverts" appears in the main nav because the route `/adverts` exists. If we decided we only wanted adverts to be reachable via their categories, that route would not be needed.

Hobo generates routes automatically by inspecting the controllers. Here's what the adverts controller looks like:

File: `app/controllers/adverts_controller.rb`

    class AdvertsController < ApplicationController

      hobo_model_controller

      auto_actions :all

    end
{: .ruby}
    
There's a couple of things to note here. Firstly, the `hobo_model_controller` declaration upgrades this controller with Hobo features. We could specify the model that this controller looks after by passing the model to `hobo_model_controller`:

    hobo_model_controller Advert
{: .ruby}
    
But we don't need to because by default Hobo infers the model from the name of the controller class.

The line `auto_actions :all` causes the standard set of RESTful actions to be added to our class: index, show, edit, create, update and destroy (if the model had any `has_many` associations there would be some actions created for those too).

The `auto_actions` declaration can be given an `:except` clause to eliminate specific actions. Try modifying that line to:

File: `app/controllers/adverts_controller.rb`

    auto_actions :all, :except => :index
{: .ruby}
    
and then refreshing the browser. You should see the "Adverts" link disappear from the main nav (and from the front page too).

### Removing a 'new' page

Make sure you're logged in to the app as 'admin', and click on "Categories" in the nav-bar. You'll see a "New Category" link at the bottom of the page. That leads to a page with a simple form for creating a new category.

The "New Category" page is a bit overkill -- a whole page for a form with a single field? It might be better to have an inline form on the main "Categories" page.

Step one is to remove the "New Category" page. Modify the `auto_actions` in the Categories controller, much as we did with the Adverts controller:

File: `app/controllers/categories_controller.rb`

    auto_actions :all, :except => :new
{: .ruby}
    
There is no step two :-) The automatic index page detects the absence of a new page and gives us an in-line form.
You'll need to restart the server for this change to take affect because Hobo's automatic routes are only loaded at
start time, even in development mode.

The categories index page should now look like this:

<img src="/images/tutorial/removing_new_page.png"/>


---

### Moving on

OK that's probably about as far as we can go with refining this app while sticking with the fully automatic UI. If this app was expected to have a short life-span or small number of users, you might well decide that the current UI is good enough. Congratulations! You just created a finished web-app with just a handful of lines of code. For apps that you want to take up a level though, you're going to want to hand-tailor the UI. That's where DRYML comes in, which is the topic of the next chapter.

Next: [Part 3 - Customizing Views with DRYML](/pod-tutorial/3-customizing-views-with-dryml)