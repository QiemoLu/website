####思路  
markdown功能的调用在django中属于自定义标签和过滤器，要先安装markdown2模块，再写一个标签库，里面定义过滤器的方法。在模板中调用就可以了。 
####写标签库和过滤方法  
    pip3 install markdown2
根目录新建templatetags,添加__init__.py使其识别为库，新建markdown.py，添加内容  

    import markdown2
    from django import template
    from django.template.defaultfilters import stringfilter
    from django.utils.encoding import force_text
    from django.utils.safestring import mark_safe
    
    register = template.Library()
    
    
    @register.filter(is_safe=True)
    @stringfilter
    # 过滤方法
    def custom_markdown(value):
        return mark_safe(markdown2.markdown(force_text(value), extras=(
            "fenced-code-blocks",
            "cuddled-lists",
            "metadata",
            "tables",
            "spoiler")))

通过{% load %}加载标签库，{{content|costom_markdown}}使用过滤器，就成功了。  

####添加代码高亮  
使用一个js脚本（highlight.js）会自动识别代码语言，进行高亮。

    <link rel="stylesheet" type="text/css" href="{% static "css/default.min.css" %}">
    <script type="text/javascript" src="{% static "js/highlight.min.js" %}"></script>
    <script>hljs.initHighlightingOnLoad();</script>  


