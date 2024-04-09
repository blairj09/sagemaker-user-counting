# RStudio on SageMaker User Counting


This demonstrates how to use the PAWs R package to query SageMaker user
details and determine how many users have access to RStudio on
SageMaker. While this doesn’t necessarily directly detail how many users
**have** accessed RStudio, it does provide detail on how many users
**can** access RStudio. This is helpful when multiple SageMaker domains
are setup and you want to know how many users have access to RStudio
across all domains.

### Setup

Load the `paws` package.

``` r
library(paws)
```

Create the `sagemaker` client. This is where you specify the profile /
credentials to use. In this case, we’re using the `example` profile
defined in our [AWS config
file](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).

``` r
sagemaker <- paws::sagemaker(credentials = list(profile = "pm-account"))
```

### User Counting

Query the list of users using `$list_user_profiles()`:

``` r
users <- sagemaker$list_user_profiles()
```

In order to retrieve user details, we need the `UserProfileName` and
`DomainId` for each user. We’ll iterate over the `users` list and use
`$describe_usr_profile()` to get the details for each user:

``` r
all_user_details <- lapply(users$UserProfiles, function(profile) {
  sagemaker$describe_user_profile(UserProfileName = profile$UserProfileName, DomainId = profile$DomainId)
})
```

Using `user_details`, we can use `UserSettings` to determine if the user
has access to RStudio. First, we’ll define a function to extract and
format details about user RStudio access:

``` r
extract_rstudio_status <- function(user_details) {
  domain_id = user_details$DomainId
  user_profile_name = user_details$UserProfileName
  user_profile_arn = user_details$UserProfileArn
  rstudio_status = user_details$UserSettings$RStudioServerProAppSettings$AccessStatus
  rstudio_usergroup = user_details$UserSettings$RStudioServerProAppSettings$UserGroup

  tibble::tibble(
    domain_id = domain_id,
    user_profile_name = user_profile_name,
    user_profile_arn = user_profile_arn,
    rstudio_status = ifelse(length(rstudio_status) == 0, NA, rstudio_status),
    rstudio_usergroup = ifelse(length(rstudio_usergroup) == 0, NA, rstudio_usergroup)
  )
}
```

We can then use this function to extract RStudio access details for each
user and combine them into a single data frame:

``` r
reduce_rbind <- \(x) Reduce(rbind, x)

sm_user_rstudio_status <- lapply(all_user_details, extract_rstudio_status) |> 
  reduce_rbind()

sm_user_rstudio_status
```

    # A tibble: 5 × 5
      domain_id  user_profile_name user_profile_arn rstudio_status rstudio_usergroup
      <chr>      <chr>             <chr>            <chr>          <chr>            
    1 d-rgw8uwd… test3             arn:aws:sagemak… ENABLED        R_STUDIO_USER    
    2 d-rgw8uwd… test2             arn:aws:sagemak… ENABLED        R_STUDIO_ADMIN   
    3 d-rgw8uwd… test1             arn:aws:sagemak… DISABLED       <NA>             
    4 d-rgw8uwd… james-admin       arn:aws:sagemak… ENABLED        R_STUDIO_ADMIN   
    5 d-rgw8uwd… james             arn:aws:sagemak… <NA>           <NA>             