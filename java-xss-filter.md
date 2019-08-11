---
title: XSS攻击过滤处理
date: 2017-12-14 10:28:03
tags: [java]
categories: 学习笔记
---

### 关于XSS攻击
XSS是一种经常出现在web应用中的计算机安全漏洞，它允许恶意web用户将代码植入到提供给其它用户使用的页面中。

<!--more-->

### XSS漏洞的危害
1. 网络钓鱼，包括盗取各类用户账号； 
2. 窃取用户cookies资料，从而获取用户隐私信息，或利用用户身份进一步对网站执行操作； 
3. 劫持用户（浏览器）会话，从而执行任意操作，例如进行非法转账、强制发表日志、发送电子邮件等； 
4. 强制弹出广告页面、刷流量等； 
5. 网页挂马； 
6. 进行恶意操作，例如任意篡改页面信息、删除文章等；
7. 进行大量的客户端攻击，如DDoS攻击；
8. 获取客户端信息，例如用户的浏览历史、真实IP、开放端口等；
9. 控制受害者机器向其他网站发起攻击；
10. 结合其他漏洞，如CSRF漏洞，实施进一步作恶；
11. 提升用户权限，包括进一步渗透网站； 
12. 传播跨站脚本蠕虫等；   
……

### 避免XSS攻击
XSS类似SQL 注入攻击，都是通过客户端恶意植入，一样也是用过滤器进行过滤。

框架使用的是SpringMVC。

#### XSS过滤器
```java
package com.devframe.filter;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

/**
 * XSS过滤器
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/12/14 10:00</pre>
 */
public class XssFilter implements Filter {

    @Override
    public void init(FilterConfig config) {
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        XssHttpServletRequestWrapper xssRequest = new XssHttpServletRequestWrapper(
                (HttpServletRequest) request);
        chain.doFilter(xssRequest, response);
    }

    @Override
    public void destroy() {
    }

}

```

#### request进行XSS过滤
```java
package com.devframe.filter;

import org.apache.commons.io.IOUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.util.StringUtils;

import javax.servlet.ReadListener;
import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * XSS过滤处理
 * @author Zhang Kai
 * @version 1.0
 * @since <pre>2017/12/14 9:33</pre>
 */
public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {
    /**
     * 没被包装过的HttpServletRequest（特殊场景，需要自己过滤）
     */
    private HttpServletRequest orgRequest;
    /**
     * html过滤
     */
    private final static HTMLFilter HTML_FILTER = new HTMLFilter();
    /**
     * Constructs a request object wrapping the given request.
     *
     * @param request request
     * @throws IllegalArgumentException if the request is null
     */
    XssHttpServletRequestWrapper(HttpServletRequest request) throws IllegalArgumentException {
        super(request);
        orgRequest = request;
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        //非json类型，直接返回
        if(!super.getHeader(HttpHeaders.CONTENT_TYPE).equalsIgnoreCase(MediaType.APPLICATION_JSON_VALUE)){
            return super.getInputStream();
        }

        //为空，直接返回
        String json = IOUtils.toString(super.getInputStream(), "utf-8");
        if (!StringUtils.hasText(json)) {
            return super.getInputStream();
        }

        //xss过滤
        json = xssEncode(json);
        final ByteArrayInputStream bis = new ByteArrayInputStream(json.getBytes("utf-8"));
        return new ServletInputStream() {
            @Override
            public boolean isFinished() {
                return true;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(ReadListener readListener) {
            }

            @Override
            public int read() {
                return bis.read();
            }
        };
    }

    /**
     * 过滤参数
     * @param name 参数name，也要过滤
     * @return String value
     */
    @Override
    public String getParameter(String name) {
        String value = super.getParameter(xssEncode(name));
        if (StringUtils.hasText(value)) {
            value = xssEncode(value);
        }
        return value;
    }

    /**
     * 过滤参数，值为数组
     * @param name 参数名
     * @return String[]
     */
    @Override
    public String[] getParameterValues(String name) {
        String[] parameters = super.getParameterValues(name);
        if (parameters == null || parameters.length == 0) {
            return null;
        }

        for (int i = 0; i < parameters.length; i++) {
            parameters[i] = xssEncode(parameters[i]);
        }
        return parameters;
    }

    /**
     * 过滤参数，返回键值对形式的参数类型
     * @return Map
     */
    @Override
    public Map<String,String[]> getParameterMap() {
        Map<String,String[]> map = new LinkedHashMap<>();
        Map<String,String[]> parameters = super.getParameterMap();
        for (String key : parameters.keySet()) {
            String[] values = parameters.get(key);
            for (int i = 0; i < values.length; i++) {
                values[i] = xssEncode(values[i]);
            }
            map.put(key, values);
        }
        return map;
    }

    /**
     * 获取request的头属性，并且进行xss过滤，返回它的值
     * @param name 属性名
     * @return String 值
     */
    @Override
    public String getHeader(String name) {
        String value = super.getHeader(xssEncode(name));
        if (StringUtils.hasText(value)) {
            value = xssEncode(value);
        }
        return value;
    }

    private String xssEncode(String input) {
        return HTML_FILTER.filter(input);
    }

    /**
     * 获取最原始的request
     * @return HttpServletRequest 原始的request
     */
    public HttpServletRequest getOrgRequest() {
        return orgRequest;
    }

    /**
     * 获取最原始的request，明确不进行xss过滤的
     * @param request request
     * @return HttpServletRequest 原始request
     */
    public static HttpServletRequest getOrgRequest(HttpServletRequest request) {
        if (request instanceof XssHttpServletRequestWrapper) {
            return ((XssHttpServletRequestWrapper) request).getOrgRequest();
        }
        return request;
    }
}

```

#### XSS过滤工具类
这个是网上看到直接copy过来，肯定比自己写的全面多了，好东西。
```java
package com.devframe.filter;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.logging.Logger;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 *
 * HTML filtering utility for protecting against XSS (Cross Site Scripting).
 *
 * This code is licensed LGPLv3
 *
 * This code is a Java port of the original work in PHP by Cal Hendersen.
 * http://code.iamcal.com/php/lib_filter/
 *
 * The trickiest part of the translation was handling the differences in regex handling
 * between PHP and Java.  These resources were helpful in the process:
 *
 * http://java.sun.com/j2se/1.4.2/docs/api/java/util/regex/Pattern.html
 * http://us2.php.net/manual/en/reference.pcre.pattern.modifiers.php
 * http://www.regular-expressions.info/modifiers.html
 *
 * A note on naming conventions: instance variables are prefixed with a "v"; global
 * constants are in all caps.
 *
 * Sample use:
 * String input = ...
 * String clean = new HTMLFilter().filter( input );
 *
 * The class is not thread safe. Create a new instance if in doubt.
 *
 * If you find bugs or have suggestions on improvement (especially regarding
 * performance), please contact us.  The latest version of this
 * source, and our contact details, can be found at http://xss-html-filter.sf.net
 *
 * @author Joseph O'Connell
 * @author Cal Hendersen
 * @author Michael Semb Wever
 */
public final class HTMLFilter {

    /** regex flag union representing /si modifiers in php **/
    private static final int REGEX_FLAGS_SI = Pattern.CASE_INSENSITIVE | Pattern.DOTALL;
    private static final Pattern P_COMMENTS = Pattern.compile("<!--(.*?)-->", Pattern.DOTALL);
    private static final Pattern P_COMMENT = Pattern.compile("^!--(.*)--$", REGEX_FLAGS_SI);
    private static final Pattern P_TAGS = Pattern.compile("<(.*?)>", Pattern.DOTALL);
    private static final Pattern P_END_TAG = Pattern.compile("^/([a-z0-9]+)", REGEX_FLAGS_SI);
    private static final Pattern P_START_TAG = Pattern.compile("^([a-z0-9]+)(.*?)(/?)$", REGEX_FLAGS_SI);
    private static final Pattern P_QUOTED_ATTRIBUTES = Pattern.compile("([a-z0-9]+)=([\"'])(.*?)\\2", REGEX_FLAGS_SI);
    private static final Pattern P_UNQUOTED_ATTRIBUTES = Pattern.compile("([a-z0-9]+)(=)([^\"\\s']+)", REGEX_FLAGS_SI);
    private static final Pattern P_PROTOCOL = Pattern.compile("^([^:]+):", REGEX_FLAGS_SI);
    private static final Pattern P_ENTITY = Pattern.compile("&#(\\d+);?");
    private static final Pattern P_ENTITY_UNICODE = Pattern.compile("&#x([0-9a-f]+);?");
    private static final Pattern P_ENCODE = Pattern.compile("%([0-9a-f]{2});?");
    private static final Pattern P_VALID_ENTITIES = Pattern.compile("&([^&;]*)(?=(;|&|$))");
    private static final Pattern P_VALID_QUOTES = Pattern.compile("(>|^)([^<]+?)(<|$)", Pattern.DOTALL);
    private static final Pattern P_END_ARROW = Pattern.compile("^>");
    private static final Pattern P_BODY_TO_END = Pattern.compile("<([^>]*?)(?=<|$)");
    private static final Pattern P_XML_CONTENT = Pattern.compile("(^|>)([^<]*?)(?=>)");
    private static final Pattern P_STRAY_LEFT_ARROW = Pattern.compile("<([^>]*?)(?=<|$)");
    private static final Pattern P_STRAY_RIGHT_ARROW = Pattern.compile("(^|>)([^<]*?)(?=>)");
    private static final Pattern P_AMP = Pattern.compile("&");
    private static final Pattern P_QUOTE = Pattern.compile("<");
    private static final Pattern P_LEFT_ARROW = Pattern.compile("<");
    private static final Pattern P_RIGHT_ARROW = Pattern.compile(">");
    private static final Pattern P_BOTH_ARROWS = Pattern.compile("<>");

    // @xxx could grow large... maybe use sesat's ReferenceMap
    private static final ConcurrentMap<String,Pattern> P_REMOVE_PAIR_BLANKS = new ConcurrentHashMap<String, Pattern>();
    private static final ConcurrentMap<String,Pattern> P_REMOVE_SELF_BLANKS = new ConcurrentHashMap<String, Pattern>();

    /** set of allowed html elements, along with allowed attributes for each element **/
    private final Map<String, List<String>> vAllowed;
    /** counts of open tags for each (allowable) html element **/
    private final Map<String, Integer> vTagCounts = new HashMap<String, Integer>();

    /** html elements which must always be self-closing (e.g. "<img />") **/
    private final String[] vSelfClosingTags;
    /** html elements which must always have separate opening and closing tags (e.g. "<b></b>") **/
    private final String[] vNeedClosingTags;
    /** set of disallowed html elements **/
    private final String[] vDisallowed;
    /** attributes which should be checked for valid protocols **/
    private final String[] vProtocolAtts;
    /** allowed protocols **/
    private final String[] vAllowedProtocols;
    /** tags which should be removed if they contain no content (e.g. "<b></b>" or "<b />") **/
    private final String[] vRemoveBlanks;
    /** entities allowed within html markup **/
    private final String[] vAllowedEntities;
    /** flag determining whether comments are allowed in input String. */
    private final boolean stripComment;
    private final boolean encodeQuotes;
    private boolean vDebug = false;
    /**
     * flag determining whether to try to make tags when presented with "unbalanced"
     * angle brackets (e.g. "<b text </b>" becomes "<b> text </b>").  If set to false,
     * unbalanced angle brackets will be html escaped.
     */
    private final boolean alwaysMakeTags;

    /** Default constructor.
     *
     */
    public HTMLFilter() {
        vAllowed = new HashMap<>();

        final ArrayList<String> a_atts = new ArrayList<String>();
        a_atts.add("href");
        a_atts.add("target");
        vAllowed.put("a", a_atts);

        final ArrayList<String> img_atts = new ArrayList<String>();
        img_atts.add("src");
        img_atts.add("width");
        img_atts.add("height");
        img_atts.add("alt");
        vAllowed.put("img", img_atts);

        final ArrayList<String> no_atts = new ArrayList<String>();
        vAllowed.put("b", no_atts);
        vAllowed.put("strong", no_atts);
        vAllowed.put("i", no_atts);
        vAllowed.put("em", no_atts);

        vSelfClosingTags = new String[]{"img"};
        vNeedClosingTags = new String[]{"a", "b", "strong", "i", "em"};
        vDisallowed = new String[]{};
        vAllowedProtocols = new String[]{"http", "mailto", "https"}; // no ftp.
        vProtocolAtts = new String[]{"src", "href"};
        vRemoveBlanks = new String[]{"a", "b", "strong", "i", "em"};
        vAllowedEntities = new String[]{"amp", "gt", "lt", "quot"};
        stripComment = true;
        encodeQuotes = true;
        alwaysMakeTags = true;
    }

    /** Set debug flag to true. Otherwise use default settings. See the default constructor.
     *
     * @param debug turn debug on with a true argument
     */
    public HTMLFilter(final boolean debug) {
        this();
        vDebug = debug;

    }

    /** Map-parameter configurable constructor.
     *
     * @param conf map containing configuration. keys match field names.
     */
    public HTMLFilter(final Map<String,Object> conf) {

        assert conf.containsKey("vAllowed") : "configuration requires vAllowed";
        assert conf.containsKey("vSelfClosingTags") : "configuration requires vSelfClosingTags";
        assert conf.containsKey("vNeedClosingTags") : "configuration requires vNeedClosingTags";
        assert conf.containsKey("vDisallowed") : "configuration requires vDisallowed";
        assert conf.containsKey("vAllowedProtocols") : "configuration requires vAllowedProtocols";
        assert conf.containsKey("vProtocolAtts") : "configuration requires vProtocolAtts";
        assert conf.containsKey("vRemoveBlanks") : "configuration requires vRemoveBlanks";
        assert conf.containsKey("vAllowedEntities") : "configuration requires vAllowedEntities";

        vAllowed = Collections.unmodifiableMap((HashMap<String, List<String>>) conf.get("vAllowed"));
        vSelfClosingTags = (String[]) conf.get("vSelfClosingTags");
        vNeedClosingTags = (String[]) conf.get("vNeedClosingTags");
        vDisallowed = (String[]) conf.get("vDisallowed");
        vAllowedProtocols = (String[]) conf.get("vAllowedProtocols");
        vProtocolAtts = (String[]) conf.get("vProtocolAtts");
        vRemoveBlanks = (String[]) conf.get("vRemoveBlanks");
        vAllowedEntities = (String[]) conf.get("vAllowedEntities");
        stripComment =  conf.containsKey("stripComment") ? (Boolean) conf.get("stripComment") : true;
        encodeQuotes = conf.containsKey("encodeQuotes") ? (Boolean) conf.get("encodeQuotes") : true;
        alwaysMakeTags = conf.containsKey("alwaysMakeTags") ? (Boolean) conf.get("alwaysMakeTags") : true;
    }

    private void reset() {
        vTagCounts.clear();
    }

    private void debug(final String msg) {
        if (vDebug) {
            Logger.getAnonymousLogger().info(msg);
        }
    }

    //---------------------------------------------------------------
    // my versions of some PHP library functions
    public static String chr(final int decimal) {
        return String.valueOf((char) decimal);
    }

    public static String htmlSpecialChars(final String s) {
        String result = s;
        result = regexReplace(P_AMP, "&amp;", result);
        result = regexReplace(P_QUOTE, "&quot;", result);
        result = regexReplace(P_LEFT_ARROW, "&lt;", result);
        result = regexReplace(P_RIGHT_ARROW, "&gt;", result);
        return result;
    }

    //---------------------------------------------------------------
    /**
     * given a user submitted input String, filter out any invalid or restricted
     * html.
     *
     * @param input text (i.e. submitted by a user) than may contain html
     * @return "clean" version of input, with only valid, whitelisted html elements allowed
     */
    public String filter(final String input) {
        reset();
        String s = input;

        debug("************************************************");
        debug("              INPUT: " + input);

        s = escapeComments(s);
        debug("     escapeComments: " + s);

        s = balanceHTML(s);
        debug("        balanceHTML: " + s);

        s = checkTags(s);
        debug("          checkTags: " + s);

        s = processRemoveBlanks(s);
        debug("processRemoveBlanks: " + s);

        s = validateEntities(s);
        debug("    validateEntites: " + s);

        debug("************************************************\n\n");
        return s;
    }

    public boolean isAlwaysMakeTags(){
        return alwaysMakeTags;
    }

    public boolean isStripComments(){
        return stripComment;
    }

    private String escapeComments(final String s) {
        final Matcher m = P_COMMENTS.matcher(s);
        final StringBuffer buf = new StringBuffer();
        if (m.find()) {
            final String match = m.group(1); //(.*?)
            m.appendReplacement(buf, Matcher.quoteReplacement("<!--" + htmlSpecialChars(match) + "-->"));
        }
        m.appendTail(buf);

        return buf.toString();
    }

    private String balanceHTML(String s) {
        if (alwaysMakeTags) {
            //
            // try and form html
            //
            s = regexReplace(P_END_ARROW, "", s);
            s = regexReplace(P_BODY_TO_END, "<$1>", s);
            s = regexReplace(P_XML_CONTENT, "$1<$2", s);

        } else {
            //
            // escape stray brackets
            //
            s = regexReplace(P_STRAY_LEFT_ARROW, "&lt;$1", s);
            s = regexReplace(P_STRAY_RIGHT_ARROW, "$1$2&gt;<", s);

            //
            // the last regexp causes '<>' entities to appear
            // (we need to do a lookahead assertion so that the last bracket can
            // be used in the next pass of the regexp)
            //
            s = regexReplace(P_BOTH_ARROWS, "", s);
        }

        return s;
    }

    private String checkTags(String s) {
        Matcher m = P_TAGS.matcher(s);

        final StringBuffer buf = new StringBuffer();
        while (m.find()) {
            String replaceStr = m.group(1);
            replaceStr = processTag(replaceStr);
            m.appendReplacement(buf, Matcher.quoteReplacement(replaceStr));
        }
        m.appendTail(buf);

        s = buf.toString();

        // these get tallied in processTag
        // (remember to reset before subsequent calls to filter method)
        for (String key : vTagCounts.keySet()) {
            for (int ii = 0; ii < vTagCounts.get(key); ii++) {
                s += "</" + key + ">";
            }
        }

        return s;
    }

    private String processRemoveBlanks(final String s) {
        String result = s;
        for (String tag : vRemoveBlanks) {
            if(!P_REMOVE_PAIR_BLANKS.containsKey(tag)){
                P_REMOVE_PAIR_BLANKS.putIfAbsent(tag, Pattern.compile("<" + tag + "(\\s[^>]*)?></" + tag + ">"));
            }
            result = regexReplace(P_REMOVE_PAIR_BLANKS.get(tag), "", result);
            if(!P_REMOVE_SELF_BLANKS.containsKey(tag)){
                P_REMOVE_SELF_BLANKS.putIfAbsent(tag, Pattern.compile("<" + tag + "(\\s[^>]*)?/>"));
            }
            result = regexReplace(P_REMOVE_SELF_BLANKS.get(tag), "", result);
        }

        return result;
    }

    private static String regexReplace(final Pattern regex_pattern, final String replacement, final String s) {
        Matcher m = regex_pattern.matcher(s);
        return m.replaceAll(replacement);
    }

    private String processTag(final String s) {
        // ending tags
        Matcher m = P_END_TAG.matcher(s);
        if (m.find()) {
            final String name = m.group(1).toLowerCase();
            if (allowed(name)) {
                if (!inArray(name, vSelfClosingTags)) {
                    if (vTagCounts.containsKey(name)) {
                        vTagCounts.put(name, vTagCounts.get(name) - 1);
                        return "</" + name + ">";
                    }
                }
            }
        }

        // starting tags
        m = P_START_TAG.matcher(s);
        if (m.find()) {
            final String name = m.group(1).toLowerCase();
            final String body = m.group(2);
            String ending = m.group(3);

            //debug( "in a starting tag, name='" + name + "'; body='" + body + "'; ending='" + ending + "'" );
            if (allowed(name)) {
                String params = "";

                final Matcher m2 = P_QUOTED_ATTRIBUTES.matcher(body);
                final Matcher m3 = P_UNQUOTED_ATTRIBUTES.matcher(body);
                final List<String> paramNames = new ArrayList<String>();
                final List<String> paramValues = new ArrayList<String>();
                while (m2.find()) {
                    paramNames.add(m2.group(1)); //([a-z0-9]+)
                    paramValues.add(m2.group(3)); //(.*?)
                }
                while (m3.find()) {
                    paramNames.add(m3.group(1)); //([a-z0-9]+)
                    paramValues.add(m3.group(3)); //([^\"\\s']+)
                }

                String paramName, paramValue;
                for (int ii = 0; ii < paramNames.size(); ii++) {
                    paramName = paramNames.get(ii).toLowerCase();
                    paramValue = paramValues.get(ii);

//          debug( "paramName='" + paramName + "'" );
//          debug( "paramValue='" + paramValue + "'" );
//          debug( "allowed? " + vAllowed.get( name ).contains( paramName ) );

                    if (allowedAttribute(name, paramName)) {
                        if (inArray(paramName, vProtocolAtts)) {
                            paramValue = processParamProtocol(paramValue);
                        }
                        params += " " + paramName + "=\"" + paramValue + "\"";
                    }
                }

                if (inArray(name, vSelfClosingTags)) {
                    ending = " /";
                }

                if (inArray(name, vNeedClosingTags)) {
                    ending = "";
                }

                if (ending == null || ending.length() < 1) {
                    if (vTagCounts.containsKey(name)) {
                        vTagCounts.put(name, vTagCounts.get(name) + 1);
                    } else {
                        vTagCounts.put(name, 1);
                    }
                } else {
                    ending = " /";
                }
                return "<" + name + params + ending + ">";
            } else {
                return "";
            }
        }

        // comments
        m = P_COMMENT.matcher(s);
        if (!stripComment && m.find()) {
            return  "<" + m.group() + ">";
        }

        return "";
    }

    private String processParamProtocol(String s) {
        s = decodeEntities(s);
        final Matcher m = P_PROTOCOL.matcher(s);
        if (m.find()) {
            final String protocol = m.group(1);
            if (!inArray(protocol, vAllowedProtocols)) {
                // bad protocol, turn into local anchor link instead
                s = "#" + s.substring(protocol.length() + 1, s.length());
                if (s.startsWith("#//")) {
                    s = "#" + s.substring(3, s.length());
                }
            }
        }

        return s;
    }

    private String decodeEntities(String s) {
        StringBuffer buf = new StringBuffer();

        Matcher m = P_ENTITY.matcher(s);
        while (m.find()) {
            final String match = m.group(1);
            final int decimal = Integer.decode(match).intValue();
            m.appendReplacement(buf, Matcher.quoteReplacement(chr(decimal)));
        }
        m.appendTail(buf);
        s = buf.toString();

        buf = new StringBuffer();
        m = P_ENTITY_UNICODE.matcher(s);
        while (m.find()) {
            final String match = m.group(1);
            final int decimal = Integer.valueOf(match, 16).intValue();
            m.appendReplacement(buf, Matcher.quoteReplacement(chr(decimal)));
        }
        m.appendTail(buf);
        s = buf.toString();

        buf = new StringBuffer();
        m = P_ENCODE.matcher(s);
        while (m.find()) {
            final String match = m.group(1);
            final int decimal = Integer.valueOf(match, 16).intValue();
            m.appendReplacement(buf, Matcher.quoteReplacement(chr(decimal)));
        }
        m.appendTail(buf);
        s = buf.toString();

        s = validateEntities(s);
        return s;
    }

    private String validateEntities(final String s) {
        StringBuffer buf = new StringBuffer();

        // validate entities throughout the string
        Matcher m = P_VALID_ENTITIES.matcher(s);
        while (m.find()) {
            final String one = m.group(1); //([^&;]*)
            final String two = m.group(2); //(?=(;|&|$))
            m.appendReplacement(buf, Matcher.quoteReplacement(checkEntity(one, two)));
        }
        m.appendTail(buf);

        return encodeQuotes(buf.toString());
    }

    private String encodeQuotes(final String s){
        if(encodeQuotes){
            StringBuffer buf = new StringBuffer();
            Matcher m = P_VALID_QUOTES.matcher(s);
            while (m.find()) {
                final String one = m.group(1); //(>|^)
                final String two = m.group(2); //([^<]+?)
                final String three = m.group(3); //(<|$)
                m.appendReplacement(buf, Matcher.quoteReplacement(one + regexReplace(P_QUOTE, "&quot;", two) + three));
            }
            m.appendTail(buf);
            return buf.toString();
        }else{
            return s;
        }
    }

    private String checkEntity(final String preamble, final String term) {

        return ";".equals(term) && isValidEntity(preamble)
                ? '&' + preamble
                : "&amp;" + preamble;
    }

    private boolean isValidEntity(final String entity) {
        return inArray(entity, vAllowedEntities);
    }

    private static boolean inArray(final String s, final String[] array) {
        for (String item : array) {
            if (item != null && item.equals(s)) {
                return true;
            }
        }
        return false;
    }

    private boolean allowed(final String name) {
        return (vAllowed.isEmpty() || vAllowed.containsKey(name)) && !inArray(name, vDisallowed);
    }

    private boolean allowedAttribute(final String name, final String paramName) {
        return allowed(name) && (vAllowed.isEmpty() || vAllowed.get(name).contains(paramName));
    }
}
```

#### web.xml配置filter
```xml
<filter>
	<filter-name>xssFilter</filter-name>
	<filter-class>com.devframe.filter.XssFilter</filter-class>
</filter>

<filter-mapping>
        <filter-name>xssFilter</filter-name>
	<url-pattern>/*</url-pattern>
</filter-mapping>
```

#### 不进行过滤的请求
只能使用servlet 的request获取原始的请求，然后 再获取参数，处理，稍微麻烦一下下。
