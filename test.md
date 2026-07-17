console.log('modal:', document.querySelectorAll('#confirmModal').length);
console.log('ok:',    document.querySelectorAll('#confirmOk').length);
document.querySelectorAll('#confirmOk').forEach((el,i)=>
  console.log(i, 'hasOnclick=', !!el.onclick, el));
console.log('bootstrap ver:', window.bootstrap && bootstrap.Modal.VERSION,
            'jqueryModal:', !!(window.jQuery && jQuery.fn.modal));