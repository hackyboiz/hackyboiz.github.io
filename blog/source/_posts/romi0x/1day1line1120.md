---
title: "[하루한줄] CVE-2024-27956: WordPress의 Automatic에서 발생하는 SQL Injection 취약점"
author: romi0x
tags: [WordPress Automatic Plugin, SQL Injection, romi0x]
categories: [1day1line]
date: 2024-11-20 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

- https://patchstack.com/articles/critical-vulnerabilities-patched-in-wordpress-automatic-plugin?_s_id=cve

## Target

- **Target Product**: WordPress Automatic Plugin (premium version by ValvePress)
- **Affected Versions**: ≤ 3.92.0
- **Patched Version**: 3.92.1

## Explain
WordPress는 사용자가 전문적인 기술 지식이 없어도 웹사이트에서 콘텐츠를 생성, 관리 및 수정할 수 있도록 도와주는 콘텐츠 관리 시스템 소프트웨어입니다. 그 중 플러그인 [Automatic](https://codecanyon.net/item/wordpress-automatic-plugin/1904470)은 WordPress에서 가장 인기 있는 자동 콘텐츠 게시 플러그인으로 알려져 있습니다.

취약점은 `inc/csv.php` 파일의 코드에 존재하며, 사용자가 전달한 `$q` 변수의 값을 SQL 쿼리로 직접 실행해 발생합니다.

`inc/csv.php` 파일의 코드 일부입니다.

```php
...

// extract query
$q = stripslashes($_POST['q']);
$auth = stripslashes($_POST['auth']);
$integ=stripslashes($_POST['integ']);

if(wp_automatic_trim($auth == '')){
	
	  echo 'login required';
	exit;
}

if(wp_automatic_trim($auth) != wp_automatic_trim($current_user->user_pass)){
	  echo 'invalid login';
	exit;
}

if(md5(wp_automatic_trim($q.$current_user->user_pass)) != $integ ){
	  echo 'Tampered query';
	exit;
}
 

$rows=$wpdb->get_results( $q);

...
```

- 우회 방법

  코드는 `$q` 변수를 SQL 쿼리로 실행하기 전에 3가지의 검증 절차를 포함하고 있지만, 우회가 가능합니다.

    1. **`$auth` 체크 우회**: `$auth` 는 빈 문자열을 허용하지 않지만, 공백(" ")을 사용하여 우회할 수 있습니다.
    2. **`$current_user->user_pass` 우회**: 인증되지 않은 사용자의 경우 `$current_user->user_pass` 값은 빈 문자열('')로 설정됩니다.
    3. **`$integ` 체크 우회**: `$integ`는 `$q` 값과 `$current_user->user_pass`를 MD5 해시한 값을 비교하는데, 인증되지 않은 사용자는 `user_pass`가 빈 문자열이므로, MD5 해시값만 맞으면 우회가 가능합니다.

그럼,  `wp_automatic_trim()` 를 봐볼까요?

```php
function wp_automatic_trim($str)
{
	if (is_null($str)) {
		return '';
	} else {
		return trim($str);
	}
}
```

함수 이름에서 알 수 있듯이 이 함수는 전달된 값에 대해 기본적인 공백 제거 작업을 수행합니다.

`null` 값을 빈 문자열로 변환하지만, 공백 문자열 자체는 처리되지 않아 우회가 가능합니다.

## Reference

- https://patchstack.com/database/vulnerability/wp-automatic/wordpress-automatic-plugin-3-92-0-unauthenticated-arbitrary-sql-execution-vulnerability?_s_id=cve