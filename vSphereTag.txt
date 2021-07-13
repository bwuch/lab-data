# This script pulls tag information from GitHub to populate into a vCenter environment. 

$webRequestTagCategories = Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/bwuch/lab-data/main/vSphereTagCategories.csv'
if ($webRequestTagCategories.StatusCode -eq 200) { 
  $vSphereTagCategories = $webRequestTagCategories.Content | ConvertFrom-CSV
} else {
  'Web Request failed'
}

$webRequest = Invoke-WebRequest -Uri 'https://raw.githubusercontent.com/bwuch/lab-data/main/vSphereTags.csv'
if ($webRequest.StatusCode -eq 200) { 
  $vSphereTags = $webRequest.Content | ConvertFrom-CSV
} else {
  'Web Request failed'
}

# todo - add some sort of logic here to exit if no tag categories/tags are returned, IE the get fails

# Get current tag categories and compare them to source
$thisTagCategory = Get-TagCategory
$tagCategoryComparison = Compare-Object ($thisTagCategory.Name) ($vSphereTagCategories.Name) -IncludeEqual

# Create missing tag categories
foreach ($missingTagCategory in ($tagCategoryComparison |?{$_.SideIndicator -eq '=>'})) {
  $thisTagDetail = $vSphereTagCategories | ?{$_.Name -eq $missingTagCategory.InputObject}
  New-TagCategory -Name $thisTagDetail.Name -Description $thisTagDetail.Description -Cardinality $thisTagDetail.Cardinality -EntityType $thisTagDetail.EntityType
}

# Confirm / Update descriptions on existing categories
foreach ($existingTagCategory in ($tagCategoryComparison |?{$_.SideIndicator -eq '=='})) {
  $thisTagDetail = $vSphereTagCategories | ?{$_.Name -eq $existingTagCategory.InputObject}
  Get-TagCategory -Name ($thisTagDetail.Name) | ?{$_.Description -ne $thisTagDetail.Description} | Set-TagCategory -Description $thisTagDetail.Description
}

# todo - we could update the EntityType, in the scenario a tag category needs to be updated to include a different object type, ie one that is configured for datastore that we want to also apply to datastore clusters after the fact.  This would be fairly easy to do, but I believe the need would be low.
# todo - consider the implications of changing the cardinality... would we really want to do this?  What if a Category has multiple today, and multiple tags are actually in use, we wouldn't be able to covert over to single without breaking tag assignments.


# Get current tags and compare them to source
$thisTag = Get-Tag
$tagComparison = Compare-Object ($thisTag.Name) ($vSphereTags.Name) -IncludeEqual

# Create missing tag categories
foreach ($missingTag in ($tagComparison |?{$_.SideIndicator -eq '=>'})) {
  $thisTagDetail = $vSphereTags | ?{$_.Name -eq $missingTag.InputObject}
  New-Tag -Name $thisTagDetail.Name -Description $thisTagDetail.Description -Category $thisTagDetail.Category
}

# Confirm / Update descriptions on existing categories
foreach ($existingTag in ($tagComparison |?{$_.SideIndicator -eq '=='})) {
  $thisTagDetail = $vSphereTags | ?{$_.Name -eq $existingTag.InputObject}
  Get-Tag -Name ($thisTagDetail.Name) | ?{$_.Description -ne $thisTagDetail.Description} | Set-Tag -Description $thisTagDetail.Description
}
