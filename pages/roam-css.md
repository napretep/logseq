---
title: roam/css
---

- [[roam/css/layout]]
	 - ```css



img{
  max-width:100%
}
code{
  padding:0px;
  margin:0px;
  font-weight:600!important;
  color:#558eda!important;
}

.roam-body-main>div, .roam-article {
    padding-right:10px !important; 
    padding-left: 10px!important;
}
.roam-block-container,.rm-block-text{
    max-width: 100% !important;
}


/* 隐藏所有rm开头的标签 */
span.rm-page-ref[data-tag*="rm"] 
{
  display: none;
}

div[data-page-links*="comment"]{
  background-color:rgba(205, 243, 220, 0.7);
}

/*
span.rm-page-ref[data-tag="comment"] 
{
  background-color:rgba(205, 243, 220, 0.7);
  
  padding:5px;
}
*/


span.rm-page-ref[data-tag="hard"]
{
  background-color:#F44336;
  color:white;
  padding:5px;
}
/* #[[rm-css-网格]] */
span.rm-page-ref[data-tag*="rm-css-网格"] 
{
  display:none;/* 这个是自己 */
  
}
div.rm-plan-expired-banner{
  display:none;
}




```

- 
