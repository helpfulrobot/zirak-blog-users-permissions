# Blog users permissions Module

A simple module for user's permission on the blog.

## Introduction

This module extends silverstripe/silverstripe-blog's module (http://addons.silverstripe.org/add-ons/silverstripe/blog).
If a lot of many writers are writing on a blog, whit this module a single user can modify and publish only his post and not the one of other users, 
with the exception of the owner, who can modify, publish and delete all BlogEntry and the administrator, who can modify, publish and delete all BlogEntry and BlogHolder. 

## Requirements

 * SilverStripe 3.1.*
 * silverstripe/blog module

## Installation

Install the module through [composer](http://getcomposer.org):

	composer zirak/blog-users-permissions

## Features
 * Each user can edit/publish/delete only his BlogEntry in the BlogHolder for which he has BLOGMANAGEMENT permissions.
 * If a user is the BlogHolder owner, he can edit/publish/delete all BlogEntry's posts, including the one writed by the administrator.
 * The administrator can edit/publish/delete all BlogHolder and BlogEntry.
 * If you create a BlogHolder with protected permissions, you can create only protected permissions BlogEntry.
 * However, in a unprotected BlogHolder, you can create protected and unprotected BlogEntry at the same time. 

## Page types

### UserRestrictedBlogEntry

These two functions populateDefaults() e getCMSFields() have the task of populate automatically the Author field with the username of who is log in to the back-end 
and put the attribute readonly, so that it can not be modified.
If you are the author of a BlogEntry, the functions canEdit() canPublish() and canDelete() return true only for your BlogEntry.
If you are the owner of a BlogHolder, the functions canEdit() canPublish() and canDelete() return true for all BlogEntry's posts.
If you are the administrators, the functions canEdit() canPublish() and canDelete() return true for all BlogEntry and BlogHolder.

```php
class UserRestrictedBlogEntry extends BlogEntry {
    
    private static $allowed_children = array('UserRestrictedBlogEntry');
    
    private static $icon = "blogUsersPermissions/images/blogpage-file.png";
    
    /*
     * This is an override of the populateDefaults() function in BlogEntry class
     * When a user create a new post, It populate the Author field with him username
     */
    public function populateDefaults(){
        parent::populateDefaults();
        
        if(Member::currentUser()){
            $username = Member::currentUser()->FirstName;
            
            if(Member::currentUser()->Surname){
                $username .= ' '.Member::currentUser()->Surname;
            }
            
            $this->owner->setField('Author', $username);
        }
    }
    
    /*
     * Add the attribute ReadOnly at the Author field in the BlogEntry form
     */
    public function getCMSFields() {
        $fields = parent::getCMSFields();
        $fields->makeFieldReadonly('Author');
        return $fields;
    }
    
    /*
     * @return boolean True if the current user can create a post.
     */
    function canCreate($member = null) {
        if(Permission::check('BLOGMANAGEMENT')){
            return true;
        }else{
            return false;
        }
    }
    
    /*
     * @return boolean True if the current user can create this post.
     */
    private function checkBlogEntryPermissions(){
        $authorsId = array();
        $sqlQuery = new SQLQuery();
        $sqlQuery->setFrom('SiteTree_versions');
        $sqlQuery->selectField('AuthorID');
        $sqlQuery->addWhere('RecordID = '.$this->ID);
        $sqlQuery->setOrderBy('ID DESC');
        $rawSQL = $sqlQuery->sql();
        $result = $sqlQuery->execute();
        foreach($result as $row) {
            $authorsId[] = $row['AuthorID'];
        }
        $sqlQuery->setDelete(true);
        
        
        if((in_array(Member::currentUser()->ID, $authorsId)) || ($this->parent->OwnerID == Member::currentUser()->ID) || (Permission::check('ADMIN'))){
            return true;
        }else{
            return false;
        }
    }
    
    /*
     * @return boolean True if the current user can edit this post.
     * Only the Administrator can edit every post, otherwise the user can edit only his posts.
     */
    function canEdit($member = null) {
        return $this->checkBlogEntryPermissions();
    }
    
    /*
     * @return boolean True if the current user can publish this post.
     * Only the Administrator can publish every post, otherwise the user can publish only his posts.
     */
    function canPublish($member = null) {
        return $this->checkBlogEntryPermissions();
    }
    
    /*
     * @return boolean True if the current user can delete this post.
     * Only the Administrator can delete every post, otherwise the user can delete only his posts.
     */
    function canDelete($member = null) {
        return $this->checkBlogEntryPermissions();
    }
    
}
```

### UserRestrictedBlogHolder

This class is similar to UserRestrictedBlogEntry, but manage the BlogHolder informations.

```php
class UserRestrictedBlogHolder extends BlogHolder {
    
    private static $icon = "blogUsersPermissions/images/blogholder-file.png";
    
    private static $allowed_children = array(
		'UserRestrictedBlogEntry'
    );
    
    /*
     * @return boolean True if the current user can edit/delete the blog
     */
    private function checkBlogHolderPermissions(){
        $authorId = DB::query('SELECT AuthorID FROM SiteTree_versions WHERE RecordID = '.$this->ID.' ORDER BY ID DESC LIMIT 1')->value();
        
        if(($authorId == Member::currentUser()->ID) || ($this->OwnerID == Member::currentUser()->ID) || (Permission::check('ADMIN'))){
            return true;
        }else{
            return false;
        }
    }
    
    /*
     * @return boolean True if the current user can edit this blog.
     * Only the Administrator can edit every blog, otherwise the user can edit only his blog.
     */
    function canEdit($member = null) {
       return $this->checkBlogHolderPermissions();
    }
    
    /*
     * $user_groups = List of group IDs to which the member belongs
     * $page_groups = List of groups that can edit this page
     * 
     * @return boolean True if the current user can add posts in this blog.
     */
    function canAddChildren($member = null) {
        $page_permission = false;
        
        $user_groups = Permission::groupList(Member::currentUser()->ID);
        
        foreach($this->EditorGroups() as $key => $group){
            foreach($user_groups as $user_group){
                if($user_group == $group->ID){
                    $page_permission = true;
                    break;
                }
            }
        }
        
        if(Permission::check('BLOGMANAGEMENT') && $page_permission){
            return true;
        }else{
            return false;
        }
    }
    
}
```