title: '我的shiro之旅: 十一 shiro的权限设计'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 11:36:00
---
在这里，介绍一个简单，基本的权限设计。其中包括3个类，有User,Role,Auth。下面是类信息:

```java

import java.io.Serializable;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

import com.concom.lang.helper.PasswordEncoder;
/**
 * @author Dylan
 * @time 2013-8-5
 */
public class User implements Serializable {
	/**
	 * 
	 */
	private static final long serialVersionUID = -5394552769206983498L;

	private Long id;
	/**
	 * 用户名
	 */
	private String username;
	/**
	 * 密码
	 */
	private String password;

	/**
	 * 角色
	 */
	private Set<Role> roles = new HashSet<Role>();
	/**
	 * 权限
	 */
	private Set<Auth> auths = new HashSet<Auth>();

	/**
	 * 用于转换role
	 */
	private Map<String, Integer> roleTrans = new HashMap<String, Integer>();
	/**
	 * 用于转换auth
	 */
	private Map<String, Integer> authTrans = new HashMap<String, Integer>();

	public User() {
	}

	public Long getId() {
		return id;
	}

	public void setId(Long id) {
		this.id = id;
	}

	/**
	 * Returns the username associated with this user account;
	 * 
	 * @return the username associated with this user account;
	 */
	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	/**
	 * Returns the password for this user.
	 * 
	 * @return this user's password
	 */
	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public Set<Role> getRoles() {
		return roles;
	}

	public void setRoles(Set<Role> roles) {
		this.roles = roles;
	}

	public Map<String, Integer> getRoleTrans() {
		return roleTrans;
	}

	public void setRoleTrans(Map<String, Integer> roleTrans) {
		this.roleTrans = roleTrans;
	}

	public Set<Auth> getAuths() {
		return auths;
	}

	public void setAuths(Set<Auth> auths) {
		this.auths = auths;
	}

	public Map<String, Integer> getAuthTrans() {
		return authTrans;
	}

	public void setAuthTrans(Map<String, Integer> authTrans) {
		this.authTrans = authTrans;
	}

	public User beforeCreate() {
		translateRole();
		translateAuth();
		return this;
	}

	/**
	 * 将装有role id的Map的值注入到role中
	 */
	private void translateRole() {
		Map<String, Integer> roleIds = getRoleTrans();
		if (roleIds.isEmpty()) {
			return;
		}
		getRoles().clear();
		for (String key : roleIds.keySet()) {
			Role role = new Role();
			role.setId(roleIds.get(key));
			getRoles().add(role);
		}
	}

	/**
	 * 将装有auth id的Map的值注入到auth中
	 */
	private void translateAuth() {
		Map<String, Integer> authIds = getAuthTrans();
		if (authIds.isEmpty()) {
			return;
		}
		getAuths().clear();
		for (String key : authIds.keySet()) {
			Auth auth = new Auth();
			auth.setId(authIds.get(key));
			getAuths().add(auth);
		}
	}

	/**
	 * @return
	 */
	public Set<String> getRolesAsString() {
		Set<String> roles = new HashSet<String>();
		for (Role role : getRoles()) {
			roles.add(role.getCode());
		}
		return roles;
	}

	/**
	 * 
	 * @return
	 */
	public Set<String> getAuthAsString() {
		Set<String> auths = new HashSet<String>();
		for (Auth auth : getAuths()) {
			auths.add(auth.getCode());
		}
		return auths;
	}

	public boolean hasRoles() {
		return !getRoles().isEmpty();
	}

	public boolean hasAuths() {
		return !getAuths().isEmpty();
	}

	public User encodePassword() {
		setPassword(PasswordEncoder.encode(getPassword()));
		return this;
	}

}
```

```java

import java.io.Serializable;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * @author Dylan
 * @time 2013-8-5
 */
public class Role implements Serializable {
	/**
	 * 
	 */
	private static final long serialVersionUID = 6769720272431073142L;

	private Integer id;

	private String code;

	private String name;

	private Set<Auth> auths = new HashSet<Auth>();

	private Map<String, Integer> authTrans = new HashMap<String, Integer>();

	public Role() {
	}

	public Role(String name) {
		this.name = name;
	}

	public Integer getId() {
		return id;
	}

	public void setId(Integer id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public String getCode() {
		return code;
	}

	public void setCode(String code) {
		this.code = code;
	}

	public Set<Auth> getAuths() {
		return auths;
	}

	public void setAuths(Set<Auth> auths) {
		this.auths = auths;
	}

	public Map<String, Integer> getAuthTrans() {
		return authTrans;
	}

	public void setAuthTrans(Map<String, Integer> authTrans) {
		this.authTrans = authTrans;
	}

	public Role beforeCreate() {
		translateAuths();
		return this;
	}

	private void translateAuths() {
		Map<String, Integer> authIds = getAuthTrans();
		if (authIds.isEmpty()) {
			return;
		}
		getAuths().clear();
		for (String key : authIds.keySet()) {
			Auth auth = new Auth();
			auth.setId(authIds.get(key));
			getAuths().add(auth);
		}

	}

	/**
	 * @return
	 */
	public Set<String> getAuthsAsString() {
		Set<String> auths = new HashSet<String>();
		for (Auth auth : getAuths()) {
			auths.add(auth.getCode());
		}
		return auths;
	}

	public boolean hasAuth() {
		return !getAuths().isEmpty();
	}

}
```

```java

import java.io.Serializable;

/**
 * @author Dylan
 * @time 2013-8-5
 */
public class Auth implements Serializable{

	/**
	 * 
	 */
	private static final long serialVersionUID = 1554690122999348374L;
	
	private Integer id;
	/**
	 * 权限代码
	 */
	private String code;
	/**
	 * 权限名称
	 */
	private String name;
	
	public Auth(){}
	
	public Integer getId() {
		return id;
	}

	public void setId(Integer id) {
		this.id = id;
	}

	public String getCode() {
		return code;
	}

	public void setCode(String code) {
		this.code = code;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}
	
}
```

因为在User里面设置的roles,auths为Set类型,Role里面设置的auths为Set类型，在jsp页面编辑注入到实体里的时候会有问题，所以均添加了Trans字段和对应的方法，目的是用于转换。

前面的文章已经介绍过realm的编写。如果对realm不了解可以看看前面的文章。在realm中我们会覆写doGetAuthorizationInfo方法，shiro在第一次需要认证权限的时候会调用这个方法，并把权限信息放到缓存中。在这个方法里，我们调用了SimpleAuthorizationInfo的addRoles方法把用户的角色添加到了info中，调用addStringPermissions方法把用户的权限信息添加到info中。这样，shiro就可以从中取出角色码和权限码进行配对。下面再简单给出doGetAuthorizationInfo方法的实现:

```java

@Override  
protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {  
          
        String username = (String)getAvailablePrincipal(principals);  
        User user = userService.getByUsername(username);  
          
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();  
        Set<String> rolesAsString = user.getRolesAsString();  
        info.addRoles(rolesAsString);  
        if(user.hasAuths()){  
            info.addStringPermissions(user.getAuthAsString());  
        }  
        for(Role role : user.getRoles()){  
            info.addStringPermissions(role.getAuthsAsString());  
        }  
        return info;  
    }
```