####Simple split testing and analytics for your 4.1+ rails app

This is a simple way to set up some basic visitor stats for your rails app. 
The only requirement is that you must have rails 4.1+ and not a massive amount of
traffic. Sometimes, I run cold email campaigns for silly side projects and I want
to know-

- [ ] who clicked and when did they click it.
- [ ] what page(s) did they visit.
- [ ] what was their click path before they left the website.
- [ ] what view were they served if I'm testing different pages. 

Very basic stuff and it didn't take that long to implement so I hope it can help you with your projects.
If everything works as expected you should have somthing like this for [your dashboard](https://s3-us-west-2.amazonaws.com/fiftyfourfooty/index.png) and something like this for your [visitor journey's](https://s3-us-west-2.amazonaws.com/fiftyfourfooty/journey.png).


####Model the events using request object

Everything I want to know can be extracted from the request object created by rails for every action
in my application. I'll want to store that info for every action I care about, and show it on some sort
of analytics dashboard. 

First, I'll create the model for the visitor events.

```ruby
  rails g model AnalyticsEvent test_name:string visitor_ip_address:string path_visited:string method_called:string controller_called:string controller_action_called:string visit_time:string
````

Nothing special here just creating the model to hold all the information from a visitor request that I care about. The test_name field is to keep track of what page the visitor saw if I'm serving up different pages for a split test. These are all methods you can call on the request object from within your `applcation_controller` so lets jump to that. Be sure to run the migration.

```ruby
  bundle exec rake db:migrate
```
####Record incoming visitor request details

Now, in your `application_controller` you can start creating these analytic_events objects after they happen and it's pretty straight forward. 
Let's make the creation of our analytics objects, after any given action happens. 

 ```ruby
  class ApplicationController < ActionController::Base
    after_action :log_analytics
	
  end
 ```
 
 So, after every controller action in the application this method is going to be called, so let's write `log_analytics`.

 ```ruby
  class ApplicationController < ActionController::Base
    after_action :log_analytics
	
	private
	
	def log_analytics
      variant = request.variant.join
	      
      AnalyticsEvent.create({      
        :test_name => variant,
        :visitor_ip_address => request.remote_ip,
        :path_visited => request.url.gsub("http://",""),
        :method_called => request.method,
        :controller_called => request.path_parameters[:controller],
        :controller_action_called => request.path_parameters[:action],
        :visit_time => Time.now
      })
	end
	
  end
 ```
 
 This shouldn't work as is because the `variant` hasn't been setup. But this is the basic pattern. Create an analytics event after a controller action in your application. Let's make it so that the `request.variant.join` makes sense. 
 
 ```ruby
  class ApplicationController < ActionController::Base
    after_action :log_analytics
	before_action :detect_variant
	
	private
	
    def detect_variant
		split_tests = ["A","B","C"]
        case split_tests.sample
        when "B"
          request.variant = :shadow
        when "C"
          request.variant = :bridge        
        else
          request.variant = :control
        end
    end
	
	def log_analytics
      variant = request.variant.join
	      
      AnalyticsEvent.create({      
        :test_name => variant,
        :visitor_ip_address => request.remote_ip,
        :path_visited => request.url.gsub("http://",""),
        :method_called => request.method,
        :controller_called => request.path_parameters[:controller],
        :controller_action_called => request.path_parameters[:action],
        :visit_time => Time.now
      })
	end
	
  end
 ```
 
 Here, we're setting up the begginings of split testing. `detect_variant` randomly selects an element from the split_tests array. Then a case statement takes over to determine what variant will be served. There is more to this explained later, but for now let's just focus on the analytics part. We just needed the request variant to be set to something we created. If it doesn't make sense right now that's ok.
 
 I'm going to change the `detect_variant` method so we get the same one every time and we'll revisit this later.
 
 ```ruby
  class ApplicationController < ActionController::Base
    after_action :log_analytics
	before_action :detect_variant
	
	private
	
    def detect_variant
		split_tests = ["A","B","C"]
		# force case statement to default to control by setting it to "A"
        case split_tests[0]
        when "B"
          request.variant = :shadow
        when "C"
          request.variant = :bridge        
        else
          request.variant = :control
        end
    end
	
	def log_analytics
	  #request variant is an array and we need to convert it to a string to creat the record.
      variant = request.variant.join
	      
      AnalyticsEvent.create({
		# this will be control now always.         
        :test_name => variant,
        :visitor_ip_address => request.remote_ip,
		# this little bit of strig processing is just to make it look prettier
        :path_visited => request.url.gsub("http://",""),
        :method_called => request.method,
        :controller_called => request.path_parameters[:controller],
        :controller_action_called => request.path_parameters[:action],
        :visit_time => Time.now
      })
	end
	
  end
 ```
 
 Ok, so now every time you execute an action in your application and event should be created. To confirm this is working visit a couple pages in your applicaiton then open up your rails console and run `AnalyticsEvent.count`. This will basically return the number of actions you just committed by jumping around your application. 
 
####Create the analytics dashbaord
 
 Cool! We have details about each request being saved as records. Now, lets create a simple dashboard to view the details of all these records we're
 saving. 
 
 `rails g controller analytics_events analytics_dashboard`
 
 Create the controller to lookup the analytics records we're creating. Then hop into `views/analytics_events/` and create the file `analytics_dashboard.html.erb`
 
 Now, back in your controller grab all the events that have been happening and make them accessible to the view. 
 
 ```ruby
 class AnalyticsEventsController < ApplicationController
  
   def analytics_dashboard
     @events = AnalyticsEvent.all.order('created_at desc').to_a
   end
  
 end
 ```

Update your routes file. If you have an admin setup through devise or something put it behind your admin namespace. I'm just going to use a basic route and then http auth. So, add this line (or something similar) to your routes file. 

`get '/a/analytics_dashboard', to: 'analytics_events#analytics_dashboard'`

And, I'm going to add basic auth to my analytics dashboard because I don't want it viewable by the public. 

```ruby
class AnalyticsEventsController < ApplicationController
  # simple auth to view page
  http_basic_authenticate_with name: "admin", password: "password"
  
  def analytics_dashboard
    @events = AnalyticsEvent.all.order('created_at desc').to_a
  end
  
end
```

####Analytics dashbaord view


Ok, now lets update our view to see the data we've been collecting. I'm using bootstrap in my project but obviously you can display this any way you want.

`analytics_dashboard.html.erb`

```html
<div class="row">
	<table class="table table-hover">
        <thead>
          <tr>
            <th>Name</th>
            <th>IP Adress</th>
            <th>Page landed</th>
            <th>HTTP Method</th>
			<th>Controller Called</th>
			<th>Controller Action</th>
			<th>Happened</th>
          </tr>
        </thead>
		<tbody>
			<% @events.each do |event| %>
			     <tr>
				   <td><%= event.test_name %></td>
				   <td><%= event.visitor_ip_address %></td>
				   <td><%= event.path_visited %></td>
				   <td><%= event.method_called %></td>
				   <td><%= event.controller_called %></td>
				   <td><%= event.controller_action_called %></td>
				   <td><%= time_ago_in_words(event.visit_time) %></td>				   
				</tr>
			<% end %>
		</tbody>
	</table>
</div>
```
You should now see a table with all the request details that we have been collecting for every action in your view at `/a/analytics_dashboard`. Now, let's update that list I had in the beginning. 

- [x] who clicked and when did they click it.
- [x] what page(s) did they visit.
- [ ] what was their click path before they left the website.
- [x] what view were they served if I'm testing different pages. 

We still need to revisit serving different pages for split testing but that 3rd item I want to improve just a bit. I want to be able to click an IP adress then see the different pages that ip address visited chronologically. This gives me some idea of that visitor's journey through the web app and when they decided to stop. 

####Seeing visitor click paths

This is easy enough so let's create another simple view. First in your routes add something like 

`get '/a/analytics_dashboard/visitor_journey', to: 'analytics_events#visitor_journey', as:  'visitor_journey'`

(please choose better names :p)

Then, back in your controller make that `visitor_journey` action you're refrencing in the routes file. 

```ruby  
class AnalyticsEventsController < ApplicationController
  http_basic_authenticate_with name: "admin", password: "password"


  def analytics_dashboard
    @events = AnalyticsEvent.all.order('created_at desc').to_a
  end
  
  def visitor_journey
    @visitor_journey = AnalyticsEvent.where(visitor_ip_address: params[:query])   
  end
  
end
```

We're going to lookup only events that happened from a certain IP address. The one we click on. We'll pass in that ip address as a query parameter
when we click through to the visitor journey page. So, let's update our analytics_dashbaord view to do that. 

`analytics_dashboard.html.erb`

```html
<div class="row">
	<table class="table table-hover">
        <thead>
          <tr>
            <th>Name</th>
            <th>IP Adress</th>
            <th>Page landed</th>
            <th>HTTP Method</th>
			<th>Controller Called</th>
			<th>Controller Action</th>
			<th>Happened</th>
          </tr>
        </thead>
		<tbody>
			<% @events.each do |event| %>
			     <tr>
				   <td><%= event.test_name %></td>				 
				   <td><%= link_to(event.visitor_ip_address, visitor_journey_url(query: "#{event.visitor_ip_address}")) %></td>
				   <td><%= event.path_visited %></td>
				   <td><%= event.method_called %></td>
				   <td><%= event.controller_called %></td>
				   <td><%= event.controller_action_called %></td>
				   <td><%= time_ago_in_words(event.visit_time) %></td>				   
				</tr>
			<% end %>			
		</tbody>
	</table>
</div>
```

Here we're basically adding the ip address to the url so we can use it to look up the events
associated with the IP adress we care about to paint a visitors journey through our app. 

One thing you may have noticed by now is we're tracking events to our actual analytics pages. We don't really care about that so let's fix it in the
`applicaiton_controller`

`after_action :log_analytics, except: [:analytics_dashboard, :visitor_journey]`

Update your after action to read like this and add any controller actions you don't care about tracking here. 

Now, we need to create the `visitor_journey` view in `views/analytics_events/`

`views/analytics_events/visitor_journey.html.erb`

In this view we have access to all the visitor actions associated with the IP address we clicked. I used some basic bootstrap styling to display this

```html
<div class='row'>
<% @visitor_journey.each do |path| %>
  <div class="col-md-6 col-md-offset-3 well" style="text-align:center;">
    <h5><small><%= time_ago_in_words(path.visit_time) %> ago</small> <br><br><%= path.path_visited %> </h5>
    <br />&#9660;<br />
  </div>
<% end %>
	<div class="col-md-6 col-md-offset-3 well" style="text-align:center;">
	Bye!
	</div>
</div>
```

Ok, now we have a table of all visitor events that happen in our applicaiton and when we click on a visitor (ip address) we get a picture of how that visitor moved through our website. Sweet!

####Basic split testing plumbing

Now, let's revisit split testing. Back in your `application_controller` change back the `detect_variant` method

```ruby
    def detect_variant
		split_tests = ["A","B","C"]
        case split_tests.sample
        when "B"
          request.variant = :shadow
        when "C"
          request.variant = :bridge        
        else
          request.variant = :control
        end
    end
``` 
 
 I've changed back the method to randomly choose an element from the split tests array. This is the bare bones of split testing but works for our purposes. `shadow` and `bridge` are just names of split tests I'm running. This could be anything but it should match your views. 
 
Now that you are setting a variant, update your controller for a page you are testing to respond to it. For example, if I'm running a test on my home marketing page to see what copy or design performs better I would update something like this. 
 
 ```ruby
 
 class StaticPagesController < ApplicationController
 
  def home
   # this method alone would just render home.html.erb
  
  end
 
 end
 
```
to something like this.

 ```ruby
 
class StaticPagesController < ApplicationController
 
  def home  
   respond_to do |format|
     format.html
     format.html.shadow # corresponds to app/views/static_pages/home.html+shadow.erb <-- create this file now
     format.html.bridge # corresponds to app/views/static_pages/home.html+bridge.erb <-- create this file now
   end
  
  end
 
end
 
```

If you're worried about a variant being set and not having a corresponding view set up, don't worry rails will fall back on the default view. In this case `home.html.erb`. Let's actually clean this up a little so we're not setting a split test name on pages we aren't testing. 

 ```ruby
  class ApplicationController < ActionController::Base
    after_action :log_analytics, except: [:analytics_dashboard, :visitor_journey]
	before_action :detect_variant, only: [:home]
	
	private
	
    def detect_variant
		split_tests = ["A","B","C"]
        case split_tests.sample
        when "A"
          request.variant = :control
        when "B"
          request.variant = :shadow
        when "C"
          request.variant = :bridge        
        else
          puts "something went wrong"
        end
    end
	
	def log_analytics
	  # the variant needs to be set to something if we're not running a test on that action
      variant = request.variant.try(:join) || "no_test"
	      
      AnalyticsEvent.create({
        :test_name => variant,
        :visitor_ip_address => request.remote_ip,
        :path_visited => request.url.gsub("http://",""),
        :method_called => request.method,
        :controller_called => request.path_parameters[:controller],
        :controller_action_called => request.path_parameters[:action],
        :visit_time => Time.now
      })
	end
	
  end
 ``` 
 
 Three things happened here. 
   1. Update the before action to only set the variant test names on pages/actions we are testing. 
   2. Change back the case statement to randomly pick an element from split_tests array 
   3. In the `log_analytics` method we need to set the variant to something and when we visit a page other than home.html.erb given the code before this it would fail. So just set it to "no_test" if a visitor visits a page we're not testing. 
   
Your control page can always be "A." Since rails fails back to default if no corresponding view is found as described above, you only have to make additional views for variants and your control will be displayed by default. A little confusing because you're setting the variant to control, but you don't need to do anything beyond that i.e. **only make additional views for variants**, rails takes care of the rest. 

This is cool because when we click on a visitor IP we can display the variant test name they were shown and see (trivially) if that got them to take another action!

That's how I got some basic visitor tracking going for my rails app after being sick of implementing tracking scripts and just wanting to see a couple things easily. Worth the time if you don't want to setup an account and just kinda want to paint a picture. 

####A Cool roadmap
- Clearly define and model click paths for visitors so we can see what they did each time they were active on the site instead of one long stream.
- Make this a gem?
- Implement long polling for analytics dashboard. Right now requires a page refresh. 
- Conditionally highlight table cells to visually see stuff that's related, e.g. tests, controller actions
- Improve patterns? (I'm not so sure this is the best way to do everything but it worked for me and was fairly straightforward)

-twiz
 