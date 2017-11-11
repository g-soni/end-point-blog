---
author: Steph Skardal
gh_issue_number: 483
tags: javascript, performance, rails
title: 'Rails Optimization: Digging Deeper'
---

I recently wrote about [raw caching performance in Rails](http://blog.endpoint.com/2011/07/raw-caching-performance-in-rubyrails.html) and [advanced Rails performance techniques](http://blog.endpoint.com/2011/07/rails-optimization-advanced-techniques.html). In the latter article, I explained how to use a Rails low-level cache to store lists of **things** during the index or list request. This technique works well for list pages, but it doesn't necessarily apply to requests to an individual **thing**, or what is commonly referred to as the "show" action in Rails applications.

In my application, the "show" action loaded at ~200 ms/request with low concurrency, with the use of Rails [fragment caching](http://api.rubyonrails.org/classes/ActionController/Caching/Fragments.html). And with high concurrency, the requests shot up to around 2000 ms/request. This wasn't cutting it! So, I pursued implementing full-page caching with a follow-up AJAX request, outlined by this diagram:

<img alt="" border="0" id="BLOGGER_PHOTO_ID_5637405702799042498" src="/blog/2011/08/05/rails-optimization-digging-deeper/image-0.png" style="display:block; margin:0px auto 10px; text-align:center;cursor:pointer; cursor:hand;" width="700"/>

First, the fully-cached is loaded (quickly). Next, an AJAX request is made to retrieve access information. The access information returns a JSON object with information on whether or not there is a user, and if that user has edit access to that thing. If there is no user, the page stays as is. If there is a user, but he does not have edit permissions, the log out button is shown and the username is populated. If there is a user and he has edit permissions, the log out button is shown, the username is populated, and additional buttons requiring edit access are shown.

### The Code

To cache the full page, I use the caches_page method, and cache only on requests of HTML format (other formats are not cached):

```ruby
class ThingsController &lt; ApplicationController
 caches_page :show, :if =&gt; Proc.new { |c| c.request.format.html? }
 ...
end
```

My access level request looks something like this:

```ruby
def accessibility
  respond_to do |format|
    format.json do
      render :json =&gt; {
        :logged_in =&gt; current_user ? current_user.to_json(:only =&gt; [:id, :username]) : false,
        :can_edit =&gt; current_user ? Thing.find(params[:id]).can_edit?(current_user) : false }
    end
  end
end
```

My HTML has some bits of code sprinkled throughout it:

```nohighlight
...
&lt;a href="#" id="edit_thing" class="requires_editability"&gt;Edit&lt;/a&gt;
...
&lt;a href="#" id="my_account" class="requires_logged_in"&gt;&lt;!-- no username yet --&gt;&lt;/a&gt;
...
```

My jQuery AJAX request looks something like the code shown below. Note that I remove elements that do not apply to the current request:

```javascript
$.ajax({
  type: 'GET',
  cache: false,
  url: editability_path,  //editability_path is defined in the HTML (a JavaScript variable)
  dataType: "JSON",
  error: function(xhr){
    $('.require_editability,.require_loggged_in').remove();
  },
  success: function(results) {
    if(results.logged_in) {
      $('.require_logged_in').show();
      $('#my_account').html(results.logged_in.username);
      if(results.can_edit) {
        $('.require_editability').show();
      } else {
        $('.require_editability').remove();
      }
    } else {
      $('.require_editability,.require_loggged_in').remove();
    }
  }
});
```

And don't forget the sweeper to clear the fully cached page after edits (or other ActiveRecord callbacks):

```ruby
class ThingSweeper &lt; ActionController::Caching::Sweeper

  observe Thing

  def after_save(record)
    expire_page :controller =&gt; :things, :action =&gt; :show, :id =&gt; record.id
  end
end
```

### Additional Notes

There are some additional notes to mention:

- If a user were to hack the AJAX or JavaScript, server-side validation is still being performed when an "edit" action is submitted. In other words, if a hacker somehow enabled an edit button to show up and post an edit, a server-side response would prohibit the update because the hacker does not have appropriate accessibility.
- HTML changes were made to accommodate this caching behavior, which was a bit tricky. HTML has to handle all potential use cases (no user, user &amp; no edit access, user &amp; edit access). jQuery itself can also be used to introduce new elements per use case.
- The access level AJAX request is also hitting more low-level Rails caches: For example, the array of **things** that a user has edit permissions is cached and the cache is cleared with standard Rails sweepers. With this additional caching component, the access level AJAX request is hitting the database minimally.
- Performance optimization scenarios such as this make an argument against inline editing of resources. If there were a backend admin interface to allow editing of **things**, full-page caching would be more straight-forward to implement.

### Conclusion

With this functionality, fully cached pages are served with an average of less than 5 ms/request, and the AJAX accessibility request appears to be around 20 ms/request (although this is harder to test with simple command line tools). This is an improvement over the 200 ms/request initially implemented. Additionally, requests at a high concurrency don't bog down the system as much.