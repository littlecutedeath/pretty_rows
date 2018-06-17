# Pretty Rows

Some Ruby/Rails code examples of mine.

## Table of contents

- [Ruby - Selenium - bot](#ruby---selenium---bot)
  - [Retrieve less used proxy from db](#retrieve-less-used-proxy-from-db)
  - [Set up capybara with selenium](#set-up-capybara-with-selenium)

## Ruby - Selenium - bot

### Retrieve less used proxy from db

```ruby
class Proxy < ActiveRecord::Base

  ...

  class << self
    def less_used_active
      target_proxies =
        ActiveRecord::Base.connection.execute(
          %(
            select proxies.id, count(proxy_id) as proxy_count
            from proxies left join accounts on accounts.proxy_id = proxies.id
            where proxies.active = true
            group by proxies.id
            order by proxy_count asc, random();
          )
        ).values

      puts "WARNING: #{target_proxies.count} active proxies remain" if target_proxies.count < 10

      target_id = target_proxies&.first&.first

      target_id ? find(target_id) : nil
    end
  end

  ...

end
```

### Set up capybara with selenium

>As a result of my experience, it is better to use Firefox instead of Chrome\
>They are both has the same features, but there is no opportunity in Chrome to handle
basic auth if you use manual proxy\
>It is all right for PAC, there is no basic auth

```ruby
require 'capybara'
require 'selenium/webdriver'

# you can specify custom path for screenshots and default time to wait for response
# Capybara.save_path = [Dir.pwd, 'log', 'error_screenshots'].join('/')
# Capybara.default_max_wait_time = 20

class CustomBrowserConfiguration
  def self.setup_firefox_with(proxy_address=nil, agent=nil)
    # firefox
    Capybara.register_driver :firefox_driver do |app|
      profile = Selenium::WebDriver::Firefox::Profile.new
      profile.proxy = Selenium::WebDriver::Proxy.new(
        http: proxy_address
      ) if proxy_address
      profile['general.useragent.override'] = agent if agent

      browser_options = Selenium::WebDriver::Firefox::Options.new(profile: profile)

      # you need this line if remote server has no GUI
      browser_options.args << '--headless' if ENV['APP_ENV'] == 'production'

      Capybara::Selenium::Driver.new(app, browser: :firefox, options: browser_options)
    end

  def self.setup_chrome_with(proxy_address=nil, agent=nil)
    # chrome
    Capybara.register_driver :chrome_driver do |app|
      browser_options = Selenium::WebDriver::Chrome::Options.new

      if ENV['APP_ENV'] == 'production'
        # you need this two lines if remote server has no GUI
        browser_options.add_argument('headless')
        browser_options.add_argument('disable-gpu')
      end

      browser_options.add_argument('start-maximized')

      browser_options.add_argument("user-agent=#{agent}") if agent
      browser_options.add_argument("proxy-server=#{proxy_address}") if proxy_address

      # WARNING: I do not know what to do with basic auth !!! (if use proxy)

      Capybara::Selenium::Driver.new(app, browser: :chrome, options: browser_options)
    end
  end

  def self.get_new_session
    Capybara::Session.new(:firefox_driver)
  end

  # basic auth in Firefox - alert window
  # so we can handle it
  def self.proxy_authenticate(session_to_auth, login, password)
    alert_window = nil

    until alert_window
      begin
        sleep(2)
        alert_window = session_to_auth.driver.browser.switch_to.alert
      rescue Selenium::WebDriver::Error::NoSuchAlertError
      end
    end

    # selenium(firefox) no longer has direct feature for basic auth handling
    # so it is the only variant
    alert_window.send_keys("#{login}#{Selenium::WebDriver::Keys[:tab]}#{password}")

    alert_window.accept

    sleep(3)
  end

  def self.close_browser(session_to_close)
    return unless session_to_close.present?
    session_to_close.driver.window_handles.size.times do
      session_to_close.driver.browser.switch_to.window(session_to_close.driver.window_handles.last)
      session_to_close.driver.browser.close
    end
    session_to_close.driver.quit
    return
  rescue Selenium::WebDriver::Error::SessionNotCreatedError
    return
  end
end
```