# Request and response times at once using curl

Source from [Stack Overflow](https://stackoverflow.com/questions/18215389/how-do-i-measure-request-and-response-times-at-once-using-curl)

## How

1. Create a new file, curl-format.txt, and paste in:

    ```
        time_namelookup:  %{time_namelookup}s\n
            time_connect:  %{time_connect}s\n
        time_appconnect:  %{time_appconnect}s\n
        time_pretransfer:  %{time_pretransfer}s\n
        time_redirect:  %{time_redirect}s\n
    time_starttransfer:  %{time_starttransfer}s\n
                        ----------\n
            time_total:  %{time_total}s\n
    ```

2. Make a request: 
    
    `curl -w "@curl-format.txt" -o /dev/null -s "http://wordpress.com/"`
    
    Or on Windows, it's...
    
    `curl -w "@curl-format.txt" -o NUL -s "http://wordpress.com/"`

## What this does

`-w "@curl-format.txt"` tells cURL to use our format file

`-o /dev/null` redirects the output of the request to `/dev/null`

`-s` tells cURL not to show a progress meter

`"http://wordpress.com/"` is the URL we are requesting. Use quotes particularly if your URL has "&" query string parameters